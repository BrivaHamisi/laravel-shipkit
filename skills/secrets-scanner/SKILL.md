---
name: secrets-scanner
description: "Add an artisan command that fails when API keys, passwords, private keys or .env files are committed to git, bundled into public JavaScript, or exposed through VITE_ variables. Use when securing a Laravel repo, preventing leaked credentials, setting up secret scanning in CI or pre-push hooks, reviewing .env handling, or after a key was pasted somewhere it shouldn't be."
license: MIT
metadata:
  author: BrivaHamisi
---

# Secrets scanner

`php artisan app:scan-secrets` fails the build when a credential is about to be published. It has no dependencies: it's one artisan command.

It checks three places secrets actually leak from:

1. **Every file git tracks.** It matches known key formats: Stripe, OpenAI, Anthropic, AWS, GitHub, Slack, Google, SendGrid, npm, private keys, Laravel `APP_KEY`, and passwords inside database URLs. It also catches long values assigned to anything named secret, password, passkey, api_key, token or client_id.
2. **The built JavaScript** in `public/build/assets`, which is usually git-ignored but downloaded by every visitor.
3. **`VITE_` variables in `.env.example`** whose names look secret. Vite compiles every `VITE_` value into public JavaScript.

It also fails if any `.env` file other than `.env.example` is tracked by git.

Findings show the file, line, kind, and a **masked** value (`sk_l...(40 chars)`), so CI logs never repeat the secret.

## Agent safety

- The command runs **locally and offline**. It reads files only to detect secrets, never sends anything over the network, and prints only a masked hint (`sk_l...(40 chars)`).
- When it reports a finding, **don't open the file to show the user the value**, and don't copy it anywhere. Tell the user the file, line and kind, and let them rotate the credential.
- Never add a real secret to `KNOWN_PUBLIC_VALUES` to silence a finding. That list is only for values the provider itself publishes, such as documented sandbox keys.

## Install

1. Copy `stubs/app/Console/Commands/ScanSecrets.php.stub` to `app/Console/Commands/ScanSecrets.php`, and the test to `tests/Feature/ScanSecretsTest.php`.
2. Run `php artisan app:scan-secrets` and fix anything it reports.
3. Run it before every push and in CI. The `pre-push-checks` skill wires both.

Scan specific files with `php artisan app:scan-secrets path/one.php path/two.env`.

## What it deliberately ignores

- References rather than values: `env('X')`, `config('x.y')`, and placeholders such as `sk_live_...`, `your-key-here`, `${DB_PASSWORD}`.
- `vendor/`, `node_modules/`, `storage/`, lock files, and images, fonts and other binaries.
- Values listed in `KNOWN_PUBLIC_VALUES`, such as a provider's publicly documented sandbox key. Allow a value there, never with an inline "ignore" comment.

## When it finds something

1. **Rotate the credential first.** Anything that reached a remote, CI log, chat or ticket is compromised, and deleting the line doesn't un-publish it: git history and forks keep it.
2. Move the value to `.env`, read it through `config()`, and leave an empty key in `.env.example`.
3. If it was pushed, remove it from history (`git filter-repo`) only **after** rotating.

## Writing tests for scanners

Never put realistic secrets in test files: GitHub push protection blocks the push, and your own scanner flags the file. Generate fake values at runtime (`fakeToken()` in the stub test) and build tell-tale prefixes from pieces (`'sk_'.'live_'`, `'-----BEGIN RSA '.'PRIVATE KEY-----'`).

## Rules for keys in Laravel apps

- Read keys only in `config/*.php` via `env()`, and use `config()` everywhere else. `env()` returns null once config is cached.
- Only public identifiers (a PayPal client ID, a Stripe publishable key) may reach the browser. Secrets stay server-side.
- One key per environment. Never reuse production keys in local or staging.
- Rotate keys shared in chat, screenshots or AI prompts.
