---
name: mpesa-stk-push
description: "Accept M-Pesa payments in Laravel with Safaricom Daraja STK push (Lipa Na M-Pesa Online): send the PIN prompt to a phone, receive the callback, poll the status, and go live. Use when adding M-Pesa, Daraja, STK push, Paybill or Till payments, mobile money checkout in Kenya, or debugging M-Pesa callbacks, timestamps, passwords or result codes."
license: MIT
metadata:
  author: BrivaHamisi
---

# M-Pesa STK push (Safaricom Daraja)

Production-tested code for taking M-Pesa payments: the customer enters their phone number, gets a PIN prompt, and your app records the result from Safaricom's callback, with a status-query fallback when the callback never arrives.

## What you get

All files are in `stubs/`. Copy each into the app at the same path, **dropping the `.stub` suffix**.

| File | Purpose |
|---|---|
| `config/mpesa.php` | Keys, short code, Paybill vs Till, callback token, limits |
| `app/Support/Payments/Mpesa.php` | Daraja client: token caching, STK push, status query, phone normalising, result codes |
| `app/Http/Controllers/MpesaPaymentController.php` | Start a payment, status polling endpoint, Safaricom callback |
| `routes/mpesa.php` | The three routes. `require` it from `routes/web.php` |
| `app/Models/Payment.php`, `app/Enums/PaymentStatus.php`, migration, factory | A reusable `payments` table with safe status transitions (shared with the `paypal-checkout` skill) |
| `resources/js/mpesa-checkout.js` | Browser helper: start the payment and poll until it finishes |
| `tests/Feature/MpesaPaymentTest.php` | 18 Pest tests covering the whole flow with `Http::fake()` |

## Steps

1. Copy the stubs. If `app/Models/Payment.php` already exists from `paypal-checkout`, keep the existing one (they're identical).
2. Add `require __DIR__.'/mpesa.php';` to the end of `routes/web.php`.
3. Run `php artisan migrate`.
4. Add these keys **with empty values** to `.env.example`. The owner fills in their own `.env`:
   ```dotenv
   MPESA_ENVIRONMENT=sandbox
   MPESA_CONSUMER_KEY=
   MPESA_CONSUMER_SECRET=
   MPESA_SHORTCODE=174379
   MPESA_PASSKEY=
   MPESA_TYPE=paybill
   MPESA_CALLBACK_TOKEN=
   ```
   The owner gets sandbox keys from their own app at developer.safaricom.co.ke (the sandbox short code is `174379`, and the sandbox passkey is on the Daraja STK push simulator page). The owner generates the callback token locally with `php -r "echo bin2hex(random_bytes(20));"`.
5. Run `php artisan test --filter=Mpesa`.
6. Build the checkout UI with `payWithMpesa()` from `resources/js/mpesa-checkout.js`, or post to `route('mpesa.store')` and poll the returned `status_url` every 4 seconds.

## Agent safety

- This skill writes integration code and tests. **Never start a real payment or call the live API yourself**: the tests use `Http::fake()`, and real M-Pesa prompts only happen when a customer pays in the running app, in sandbox until the owner switches to live.
- **Never ask the user to paste keys into the chat, and never write key values into any file.** Add the variable names with empty values to `.env.example`; the owner sets the real values in their own `.env` or the host's environment settings.
- Don't read, print or copy the contents of `.env`.

## Rules that matter

- **Never trust an amount from the browser for priced things.** The controller takes `amount` from the request, which is right for donations. For orders or invoices, compute the amount on the server and attach the payment to the order through the `payable` morph relation.
- **The callback is the source of truth**, but it can't reach `localhost` and is sometimes late. The status endpoint asks Daraja directly once a payment has been pending for 20 seconds. Keep both.
- **Check the paid amount in the callback.** `CallbackMetadata.Amount` smaller than the payment means it failed, whatever the result code says.
- **Protect the callback with a secret in the URL** (`/mpesa/callback/{token}`), compared with `hash_equals`. Safaricom doesn't sign callbacks. Exempt that route from CSRF (the stub uses `withoutMiddleware(ValidateCsrfToken::class)`), and always answer `{"ResultCode":0}` so Safaricom stops retrying.
- **Completed is final.** `markCompleted()` also rescues a payment marked failed or abandoned (a stray cancel, a query that timed out), but `markFailed()` never overrides a completed payment. Duplicate callbacks are harmless.
- **Identify payments by a UUID `reference`** in URLs, never the numeric ID.

## Daraja gotchas

- **Timestamp** must be Kenya time (`Africa/Nairobi`) as `YmdHis`, whatever your app's timezone. A UTC timestamp gives "Invalid Timestamp" or a wrong password.
- **Password** = `base64(shortcode + passkey + timestamp)`, with the same timestamp sent in the request.
- **Till vs Paybill:** a Till uses `CustomerBuyGoodsOnline` with `PartyB` set to the **till number**. That differs from the store short code used as `BusinessShortCode`. A Paybill uses `CustomerPayBillOnline` with the short code in both fields.
- **Phone numbers** must be `2547XXXXXXXX` or `2541XXXXXXXX`. `Mpesa::normalizePhone()` accepts `07...`, `01...`, `+254...` and spaces.
- **Whole shillings only.** The amount is rounded to an integer.
- **`AccountReference` max 12 characters, `TransactionDesc` max 13.** Longer values are rejected, so the client truncates them.
- **Status query while the customer is still deciding** returns HTTP 500 with `errorCode` `500.001.1001`. Treat that as "still pending", not an error.
- **Access tokens last an hour.** They're cached (minus two minutes) per environment.
- **Result codes:** 0 paid, 1 insufficient balance, 1032 cancelled by user, 1037 timed out (phone unreachable), 2001 wrong PIN. `Mpesa::describeResult()` turns them into customer-friendly text.

## Testing a real prompt locally

This is for the developer to do by hand, with their own sandbox keys and phone. Safaricom must reach the callback over public HTTPS, so the developer can expose their local site through a tunnel service they already use and set `MPESA_CALLBACK_URL` to `https://<tunnel-host>/mpesa/callback/<token>`. Without a tunnel, the status-query fallback still completes sandbox payments after about 20 seconds.

## Going live

1. Get a Paybill or Till (through Safaricom or your bank) with Lipa Na M-Pesa Online enabled.
2. On the Daraja portal, create a production app and use **Go Live** to link your short code. The live passkey arrives by email.
3. Set `MPESA_ENVIRONMENT=live`, plus the live key, secret, short code, passkey and `MPESA_TYPE`.
4. Make sure the callback URL is public HTTPS and not behind basic auth or a login.
5. The owner makes one small real payment themselves and checks that the callback marked it completed with a receipt number.

## Rate limits

The stub routes use simple per-IP throttles. Mobile carriers share one IP across many customers, so pair this with the `shared-network-rate-limits` skill, which limits prompts per phone number instead.
