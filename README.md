# Shipkit: production skills for Laravel

**Shipkit is 12 Laravel skills for AI coding agents, taken from a live production site. Each one ships real code and passing Pest tests.**

Most skills are advice. Shipkit skills are working features: M-Pesa and PayPal payments, a secrets scanner, spam-proof forms, SEO files, image pipelines and WordPress migration. Each comes with the code, the tests, and the non-obvious lessons that only show up in production. CI installs every skill into a brand-new Laravel app and runs its tests, every week.

```bash
php artisan boost:add-skill BrivaHamisi/laravel-shipkit
```

Pick the skills you want, or install them all:

```bash
php artisan boost:add-skill BrivaHamisi/laravel-shipkit --all
php artisan boost:add-skill BrivaHamisi/laravel-shipkit --skill=mpesa-stk-push --skill=secrets-scanner
php artisan boost:add-skill BrivaHamisi/laravel-shipkit --list
```

Then ask your agent (Claude Code, Cursor, Copilot, Codex, Junie...): *"Add M-Pesa payments"*, *"Make link previews work on WhatsApp"*, *"Stop the spam on our contact form"*.

### Without an agent

Run this inside your Laravel project:

```bash
npx laravel-shipkit list                       # the 12 skills
npx laravel-shipkit add secrets-scanner        # one or more skills, or: all
npx laravel-shipkit add seo-crawlability --dry-run
```

It copies the code and tests into your app, wires up routes, providers and composer scripts, and prints the next steps. Files you've changed are never overwritten unless you pass `--force`. The package has no dependencies and no install scripts, and needs no network access once downloaded.

## The skills

### Payments

| Skill | What it adds | Tests |
|---|---|---|
| [`mpesa-stk-push`](skills/mpesa-stk-push/SKILL.md) | Safaricom Daraja STK push: PIN prompt, callback, status-query fallback, Paybill and Till, go-live checklist | 18 |
| [`paypal-checkout`](skills/paypal-checkout/SKILL.md) | PayPal Orders v2 Smart Buttons with server-side create and capture, and full-amount verification | 12 |

### Security and CI

| Skill | What it adds | Tests |
|---|---|---|
| [`secrets-scanner`](skills/secrets-scanner/SKILL.md) | `app:scan-secrets`: fails on committed keys (Stripe, OpenAI, Anthropic, AWS, GitHub...), tracked `.env` files, secret `VITE_` variables and keys bundled into JavaScript | 13 |
| [`pre-push-checks`](skills/pre-push-checks/SKILL.md) | `composer check:push` with a git pre-push hook and matching GitHub Actions: build, scan, lint, test | 3 |
| [`shared-network-rate-limits`](skills/shared-network-rate-limits/SKILL.md) | Per-person rate limits (email, phone, resource), so thousands of mobile users behind one carrier IP don't block each other | 6 |
| [`bot-proof-forms`](skills/bot-proof-forms/SKILL.md) | CAPTCHA-free spam protection: an encrypted time token plus a honeypot, as middleware and a Blade component | 9 |

### SEO and sharing

| Skill | What it adds | Tests |
|---|---|---|
| [`seo-crawlability`](skills/seo-crawlability/SKILL.md) | Dynamic `robots.txt` (staging never indexed), `sitemap.xml`, `llms.txt`, and a `noindex` middleware | 6 |
| [`json-ld-structured-data`](skills/json-ld-structured-data/SKILL.md) | A request-scoped schema.org `@graph`: Organization, WebSite, BlogPosting, BreadcrumbList, FAQPage. XSS-safe | 4 |
| [`social-share-previews`](skills/social-share-previews/SKILL.md) | One Blade component for Open Graph, Twitter cards, canonical and meta description tags that work on WhatsApp | 3 |

### Content and admin

| Skill | What it adds | Tests |
|---|---|---|
| [`image-uploads`](skills/image-uploads/SKILL.md) | Uploads become WebP sizes plus a JPEG share image, portrait photos keep their heads when cropped, and a regenerate command | 10 |
| [`inertia-admin-tables`](skills/inertia-admin-tables/SKILL.md) | Inertia React and shadcn/ui table pagination: page numbers, row count, a rows-per-page choice that keeps filters | 7 |
| [`wordpress-migration`](skills/wordpress-migration/SKILL.md) | WordPress SQL dump → clean JSON (posts, pages, terms, authors, featured images, UTC dates), plus spam filtering for imported form data | 14 |

**105 tests in total**, all run in CI against a fresh Laravel app.

## What makes these different

- **Working code, not just advice.** Each skill ships its files in `stubs/`, taken from a production app and generalised.
- **Tested end to end.** `bin/test-skill` copies a skill into a clean `laravel/laravel` install exactly as an agent would, then runs its Pest tests. CI does this for every skill on every push, and weekly against the newest Laravel.
- **The lessons learned the hard way are written down.** Daraja wants Kenya time in its password. WhatsApp shows no preview for WebP images. A shadcn Select at the bottom of a page opens with nothing showing. Throttling runs before route model binding. Each skill's "Rules" and "Gotchas" sections list the traps that cost someone a day.
- **Secure by default.** Server-side amounts, secret callback URLs, masked scanner output, XSS-safe JSON-LD, allow-listed page sizes. The repository scans itself with its own secrets scanner in CI.

## How a skill is laid out

```
skills/mpesa-stk-push/
├── SKILL.md          # what the agent reads: when to use it, steps, rules, gotchas
├── shipkit.json      # how the test harness wires it up (routes, providers, packages)
└── stubs/            # the code, at the paths it belongs in the app
    ├── app/Support/Payments/Mpesa.php.stub
    ├── routes/mpesa.php.stub
    └── tests/Feature/MpesaPaymentTest.php.stub
```

Code files end in `.stub` because `boost:add-skill` deliberately doesn't download `.php` files. Agents copy each stub to its path without the suffix.

## Requirements

PHP 8.2+ and Laravel 11+. CI tests against the latest Laravel release. `image-uploads` needs `intervention/image` ^3.11, and `inertia-admin-tables` expects Inertia React with shadcn/ui.

## Develop and test

```bash
bin/make-test-app                      # a fresh Laravel app in .test-app (once)
SHIPKIT_APP=.test-app bin/test-skill all
SHIPKIT_APP=.test-app bin/test-skill mpesa-stk-push
php bin/check-skills                   # frontmatter, .stub naming, shared files
npm test                               # the npx installer
```

The `payments` model, enum, migration and factory live once in `.shared/payments`. After changing them, run `bin/sync-shared`.

**Releasing:** bump `version` in `package.json`, commit, then publish a GitHub release tagged `v<version>`. The `publish` workflow tests the installer and publishes to npm with provenance through trusted publishing, so no npm token is stored anywhere.

## Contributing

Pull requests are welcome, especially skills for more payment providers (Stripe, Flutterwave, Paystack, Airtel Money, MTN MoMo) and more "every app needs this" features. A new skill needs:

1. A `SKILL.md` with `name` (matching the folder), a quoted `description` saying **what it does and when to use it**, and `license: MIT`.
2. Its code in `stubs/` as `.stub` files, with **Pest tests that pass on a fresh Laravel app**.
3. A "Rules" or "Gotchas" section with what you learned the hard way.

No real credentials, personal data or customer names in stubs or fixtures. Generate fake secrets at runtime; the scanner runs in CI.

## License

MIT. Use it, change it, ship it.
