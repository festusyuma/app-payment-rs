# payment

Library crate for taking payments and sending payouts through [Paystack](https://paystack.com), with each payment, payout and saved bank account recorded in DynamoDB so its status can be checked and updated later. It provides three services that share one `Config`: `PaymentService`, `AccountService` and `PayoutService`.

## What it does

- **Payments (`PaymentService`)**
  - **`create_payment`**: starts a Paystack checkout for an email and amount under an ID you choose (an order ID, for example). It returns Paystack's `authorization_url` to send the customer to, and records the payment as `pending`. Fails if a payment with that ID already exists.
  - **`verify_payment`**: asks Paystack for the payment's current status and stores it. Once a payment is `success`, it's returned from the table without calling Paystack again. A `success` whose amount doesn't match the recorded amount is stored as `failed`.
- **Bank accounts (`AccountService`)**
  - **`list_banks`**: the banks Paystack supports, cached for an hour.
  - **`create_recipient`**: resolves an account number and bank code to the account holder's name, registers it with Paystack as a transfer recipient, saves it, and returns a new account ID to pay out to.
  - **`get_recipient`**: looks up a saved account by ID, cached for 60 seconds.
- **Payouts (`PayoutService`)**
  - **`init_transfer`**: sends an amount from your Paystack balance to a saved account, with an optional reason. Returns the payout ID and its status.
  - **`complete_transfer`**: finalizes a payout that came back as `otp` (when OTP confirmation is enabled on your Paystack account) with the OTP Paystack sent you.
  - **`verify_transfer`**: asks Paystack for the payout's current status and stores it.
  - **`update_transfer_status`**: sets a payout's status directly, for example from a Paystack webhook.

Amounts are in the currency's smallest unit (kobo for NGN), as Paystack expects.

## Wiring it into an app

```rust
use payment::{AccountService, Config, PaymentService, PaymentStatus, PayoutService};
use std::sync::Arc;

let config = Arc::new(
    Config::builder()
        .paystack_secret("sk_live_...")
        .payment_table_name("payments")
        .accounts_table_name("payment-accounts")
        .payout_table_name("payouts")
        .build()
        .await,
);

let payments = PaymentService::new(config.clone());
let accounts = AccountService::new(config.clone());
let payouts = PayoutService::new(config);

// Take a payment: send the customer to `init.authorization_url`, then verify once they return.
let init = payments
    .create_payment("order-123", "customer@example.com", 500_000, Some("https://example.com/paid"))
    .await?;

if payments.verify_payment("order-123").await? == PaymentStatus::Success {
    // fulfil the order
}

// Pay someone out: save their bank account once, then transfer to it by ID.
let account_id = accounts.create_recipient("058", "0123456789").await?;
let payout = payouts.init_transfer(&account_id, 250_000, Some("Withdrawal")).await?;
```

Every method returns `payment::ServiceResult<T>`, which is `Result<T, shared::error::ServiceError>`, so results can be returned from `axum` handlers through `shared::response::ServiceResponse`.

## Config

Built via `Config::builder()...build().await`. It uses `aws_config::load_from_env()` for the DynamoDB client, so it needs AWS credentials and a region available the usual way (env vars, instance profile, etc.). Every setting is required, and `build()` panics if one is missing.

| Method                   | Description                                       |
| ------------------------ | ------------------------------------------------- |
| `.paystack_secret()`     | Your Paystack secret key.                         |
| `.payment_table_name()`  | DynamoDB table for payments.                      |
| `.accounts_table_name()` | DynamoDB table for saved bank accounts.           |
| `.payout_table_name()`   | DynamoDB table for payouts.                       |

Each table needs a string partition key named `id` and no sort key.

## Statuses

`PaymentStatus` and `PayoutStatus` mirror Paystack's transaction and transfer statuses. Anything Paystack returns that isn't listed maps to `Pending`.

- **`PaymentStatus`**: `Pending`, `Ongoing`, `Processing`, `Queued`, `Success`, `Failed`, `Abandoned`, `Reversed`.
- **`PayoutStatus`**: `Pending`, `Otp`, `Received`, `Processing`, `Success`, `Failed`, `Reversed`, `Abandoned`, `Blocked`, `Rejected`.

## Errors

Failures surface as `shared::error::ServiceError::HttpMessage`:

| Case | Status |
|---|---|
| A payment ID, account or payout already exists | `409` |
| A payment, account or payout isn't found | `400` |
| `complete_transfer` on a payout that isn't awaiting an OTP | `400` |
| Paystack rejects a request | `400`, with Paystack's message |
| Network, DynamoDB or parsing failures | `500` (logged) |

## Known gaps

- **NGN only.** Bank lists and transfer recipients are fixed to Nigerian banks (`currency=NGN`, `nuban` recipients).
- **No webhook handling.** Statuses only change when you call `verify_payment`/`verify_transfer`, or pass a status to `update_transfer_status` yourself.
