---
name: shared-network-rate-limits
description: "Rate limit Laravel forms, OTP/SMS/M-Pesa prompts, status polling, downloads and APIs per person (email, phone, resource, user) instead of per IP, so thousands of mobile users behind one carrier IP (CGNAT), office or campus network don't lock each other out. Use when adding throttling, RateLimiter::for, 429 Too Many Requests errors, brute-force or spam protection, or when legitimate users report being blocked."
license: MIT
metadata:
  author: BrivaHamisi
---

# Rate limits that don't punish shared networks

`throttle:5,1` counts per IP address. That is fine on a home connection and wrong almost everywhere else. Mobile carriers put thousands of phones behind a handful of public IPs (carrier-grade NAT), and so do offices, schools, campus Wi-Fi and some countries' entire mobile networks. One busy minute, and strangers use up each other's allowance.

This skill registers named limiters that **count per person first** (email, phone number, the resource being polled, the signed-in user), with a **generous per-IP backstop** that only bots reach.

| Limiter | Per person | Per IP backstop | Use on |
|---|---|---|---|
| `forms` | 3/min per email (or phone, username) per form | 30/min | contact, newsletter, sign-up, password reset |
| `phone-prompts` | 3 per 5 min per phone number | 30/min | OTP, SMS, WhatsApp, M-Pesa STK push |
| `polling` | 30/min per URL (each resource) | 600/min | payment and job status endpoints |
| `downloads` | – | 120/min | file downloads |
| `api` | 120/min per user when signed in | 120/min for guests | API routes |

## Install

1. Copy `stubs/app/Providers/RateLimitServiceProvider.php.stub` to `app/Providers/RateLimitServiceProvider.php`.
2. Register it in `bootstrap/providers.php` (Laravel 11+) or in `config/app.php` providers (Laravel 10).
3. Replace numeric throttles on routes:
   ```php
   Route::post('subscribe', ...)->middleware('throttle:forms');
   Route::post('otp/send', ...)->middleware('throttle:phone-prompts');
   Route::get('payments/{payment:reference}', ...)->middleware('throttle:polling');
   ```
4. Copy the test to `tests/Feature/RateLimitsTest.php` and run it. It proves 20 people on one IP can all subscribe, while one person is still limited.
5. Update any existing test that expected a 429 after N requests **from different people**. That was the bug.

## Make sure `$request->ip()` is the visitor's IP

Every per-IP limit silently becomes **site-wide** if the app sees the proxy's IP instead of the visitor's.

- **Served directly** (cPanel, a VPS with nginx or Apache): nothing to do.
- **Behind a load balancer, Cloudflare or a CDN**: trust the proxy, or everyone shares one limit. In `bootstrap/app.php`:
  ```php
  ->withMiddleware(function (Middleware $middleware) {
      $middleware->trustProxies(at: '*'); // or the proxy's IP ranges
  })
  ```
  With Cloudflare, prefer trusting only Cloudflare's published IP ranges. The real visitor IP is in `CF-Connecting-IP`.
- Check it: log `$request->ip()` from two different phones on mobile data. The values must differ.

## Rules

- **Limit per identity for the action, per IP only as a backstop.** Validation still runs after the limiter, so an identity-less request only gets the IP limit.
- **Normalise identities.** `Same@Example.org ` and `same@example.org` are one person. `0712 345 678` and `+254712345678` are one phone (last nine digits).
- **Key `polling` by path**, so each payment or job has its own allowance. With per-IP polling, five customers on one network checking every four seconds (75 requests a minute) would block each other.
- **Throttling runs before route model binding.** Inside a limiter, `$request->route('payment')` is still the raw string, not a model. Use `$request->route()->originalParameter('payment')` or the path.
- **Give each `Limit` in an array a distinct key prefix.** Two limits with the same key share one counter.
- **The cache store is the counter.** `database`, `redis` and `memcached` work across servers. The `file` driver only works on a single server, and `array` resets every request.
- Page views don't need throttling. Limit **actions** (submissions, sends, payments, downloads), not reading.
