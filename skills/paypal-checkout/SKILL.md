---
name: paypal-checkout
description: "Accept PayPal and card payments in Laravel with PayPal Orders v2 and Smart Buttons, where the server creates and captures every order so the amount can't be tampered with. Use when adding PayPal, PayPal buttons, card checkout through PayPal, donations, or when fixing PayPal capture, amount mismatch, duplicate order or sandbox/live issues."
license: MIT
metadata:
  author: BrivaHamisi
---

# PayPal checkout (Orders v2, server-side capture)

PayPal's Smart Buttons run in the browser, but here **the server creates and captures every order**. The browser can't change the amount, and a payment only counts when PayPal confirms the full amount in the right currency.

## What you get

Copy each file in `stubs/` to the same path in the app, **dropping the `.stub` suffix**.

| File | Purpose |
|---|---|
| `config/paypal.php` | Mode, client ID and secret, currency, amount limits |
| `app/Support/Payments/PayPal.php` | OAuth token caching, create order, capture (idempotent), `paidInFull()` check |
| `app/Http/Controllers/PayPalCheckoutController.php` | Create, capture and cancel endpoints for the buttons |
| `routes/paypal.php` | The three routes. `require` it from `routes/web.php` |
| `app/Models/Payment.php`, `app/Enums/PaymentStatus.php`, migration, factory | The shared `payments` table (identical to the `mpesa-stk-push` skill's) |
| `resources/js/paypal-checkout.js` | `renderPayPalButtons()`, wired to the endpoints |
| `tests/Feature/PayPalCheckoutTest.php` | 12 Pest tests with `Http::fake()` |

## Steps

1. Copy the stubs. If `app/Models/Payment.php` already exists from `mpesa-stk-push`, keep it.
2. Add `require __DIR__.'/paypal.php';` to `routes/web.php`, then run `php artisan migrate`.
3. Add to `.env`:
   ```dotenv
   PAYPAL_MODE=sandbox
   PAYPAL_CLIENT_ID=
   PAYPAL_CLIENT_SECRET=
   PAYPAL_CURRENCY=USD
   ```
   Create the app at developer.paypal.com → Apps & Credentials. Sandbox and live have **different** credentials.
4. Load the SDK on the checkout page with **only the client ID**:
   `https://www.paypal.com/sdk/js?client-id=...&currency=USD&intent=capture&components=buttons`
5. Call `renderPayPalButtons('#paypal-buttons', () => ({ amount }), onResult)`.
6. Run `php artisan test --filter=PayPal`.

## Rules that matter

- **The client secret stays on the server.** Only the client ID goes in the page. Never put the secret in a `VITE_` variable: Vite compiles those into public JavaScript.
- **Verify the capture, not the button callback.** `onApprove` only means the customer clicked. `PayPal::paidInFull()` checks that the capture `status` is `COMPLETED`, that the currency matches, and that the amount equals the payment's. A `PENDING` capture (under review) is not paid.
- **Idempotency:** the `PayPal-Request-Id` header (the payment's UUID) stops retries from creating duplicate orders. Capturing an already-captured order returns `ORDER_ALREADY_CAPTURED`, and the client then fetches the order instead of failing.
- **Never trust the browser for prices.** The stub takes `amount` from the request (donations). For orders, compute the amount server-side and link the payment through `payable`.
- **Cancel isn't failure.** Closing the pop-up marks the payment `abandoned`. If PayPal later confirms it anyway, `markCompleted()` still records it.
- **`shipping_preference: NO_SHIPPING`** hides the address step for donations and digital goods. Remove it if you ship things.

## Going live

1. Create a **live** app in the PayPal dashboard. It needs a verified business account.
2. Set `PAYPAL_MODE=live` with the live client ID and secret. The client picks `api-m.paypal.com` automatically.
3. Make a small real payment and refund it from the PayPal dashboard.
4. Rotate the credentials if they were ever pasted into chat, email or a commit. The `secrets-scanner` skill catches committed ones.
