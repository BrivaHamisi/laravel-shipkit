---
name: bot-proof-forms
description: "Stop spam and bot submissions on Laravel contact, newsletter, sign-up, donation and checkout forms without a CAPTCHA, using an encrypted time token plus a honeypot field in a reusable middleware and Blade component. Use when forms receive spam, fake sign-ups, card-testing or injection attempts, or when adding a public form, a honeypot, or anti-bot protection."
license: MIT
metadata:
  author: BrivaHamisi
---

# Bot-proof forms without a CAPTCHA

Two invisible checks stop most form spam, with no friction for real people and no third-party script:

1. **Time token.** An encrypted "form shown at" timestamp. People take at least a few seconds; bots post instantly or replay a form scraped days ago. Tokens younger than 3 seconds or older than 24 hours fail, and so do forged ones (they're encrypted with `APP_KEY`).
2. **Honeypot.** A `website` field hidden from people and screen readers. Only bots fill it in.

## Install

1. Copy from `stubs/`, dropping `.stub`:
   - `app/Support/FormTimeToken.php`
   - `app/Http/Middleware/SpamGuard.php`
   - `resources/views/components/spam-guard.blade.php`
   - `tests/Feature/SpamGuardTest.php`
2. Put `<x-spam-guard />` inside every public form, next to `@csrf`.
3. Protect the routes:
   ```php
   use App\Http\Middleware\SpamGuard;

   Route::post('contact', ContactController::class)->middleware([SpamGuard::class, 'throttle:forms']);
   Route::post('checkout', CheckoutController::class)->middleware(SpamGuard::class.':reject');
   ```
4. **JavaScript-submitted forms** must send `form_token` and `website` too. Read them from the rendered inputs; don't build a new token in JS.
5. In your existing form tests, send a token issued at least 3 seconds earlier:
   ```php
   'form_token' => Crypt::encryptString((string) now()->subSeconds(10)->getTimestamp()),
   ```

## Silent vs reject

- **Silent** (default): the bot gets the same success reply a person would, but the controller never runs. Bots can't tell they were caught, so they don't adapt. Use it for contact and newsletter forms.
- **Reject** (`:reject`): answers with a 422 and "please try again". Use it where a real person caught by mistake (a password manager filled the form in 2 seconds) must be told to retry: payments, sign-ups, bookings.

## Rules

- **Don't reveal who's subscribed.** The newsletter endpoint must give the same answer for new and existing emails.
- **Block header injection** in fields that end up in email headers (name, subject): `'not_regex:/[\r\n]/'`.
- **Validate after the guard.** Bots never reach validation, so they can't probe your rules.
- **Pair with per-person rate limits** (the `shared-network-rate-limits` skill). The guard stops dumb bots; limits stop persistent ones.
- **Card-testing bots** hammer donation and checkout forms with tiny amounts. Use `:reject` there, set a sensible minimum amount, and keep the provider's own fraud tools on.
- Page caching breaks the token. If a form page is fully cached (a CDN caching HTML), the token becomes old for everyone. Render the token through a small uncached request, or exclude form pages from the cache.
- Add a CAPTCHA (Turnstile, hCaptcha) only if these checks plus rate limits aren't enough. Most sites never need one.
