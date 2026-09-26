---
name: pre-push-checks
description: "Run the production build, secrets scan, Pint, static analysis and the full test suite automatically before every git push, and the same checks in GitHub Actions CI. Use when setting up git hooks, a pre-push hook, composer check scripts, CI workflows, or when broken builds, failing tests or leaked secrets keep reaching the main branch."
license: MIT
metadata:
  author: BrivaHamisi
---

# Pre-push checks

One command, `composer check:push`, runs everything that must pass before code leaves your machine. A git `pre-push` hook runs it automatically, and CI runs the identical command, so "works on my machine" and "CI is red" stop disagreeing.

## Agent safety

- **Ask the user before enabling the hook.** `composer hooks:install` changes only this repository's git config (`core.hooksPath=.githooks`). Nothing system-wide, global or outside the project is touched.
- The hook runs the project's own scripts (`composer check:push`) and nothing downloaded at runtime.

## Install

1. Copy `stubs/.githooks/pre-push.stub` to `.githooks/pre-push` and make it executable: `chmod +x .githooks/pre-push`.
2. Copy `stubs/.github/workflows/checks.yml.stub` to `.github/workflows/checks.yml`, merging it into an existing workflow if there is one.
3. Merge these scripts into `composer.json`, and adapt `check:push` to the tools the project uses:
   ```json
   "scripts": {
       "check:push": [
           "Composer\\Config::disableProcessTimeout",
           "npm run build",
           "@php artisan app:scan-secrets",
           "@php artisan config:clear --ansi",
           "pint --test",
           "@php artisan test"
       ],
       "hooks:install": [
           "@php -r \"is_dir('.git') && passthru('git config core.hooksPath .githooks');\""
       ]
   }
   ```
   Add `phpstan analyse`, `npm run lint` or `tsc --noEmit` when the project has them. `app:scan-secrets` comes from the `secrets-scanner` skill; install it too, or remove that line.
4. Run `composer hooks:install` once per clone. To make it automatic, add `"@hooks:install"` to the `post-install-cmd` script, or to the project's `setup` script.
5. Copy `stubs/tests/Feature/PrePushChecksTest.php.stub` to `tests/Feature/PrePushChecksTest.php`. It fails if someone deletes the hook, the script or the CI step.

## Why each step is there

- **`npm run build` first.** A TypeScript or import error that only shows in the production build is the most common "passed locally, broke deploy" failure. Building before tests also creates the Vite manifest that feature tests rendering pages need.
- **Secrets scan before tests,** so a leaked key blocks the push even when tests are slow or flaky.
- **`config:clear`,** so a cached config from `php artisan config:cache` can't make tests read the wrong environment.
- **`pint --test`,** not `pint`: checks must never rewrite files mid-push.

## Rules

- **Hooks live in the repo** (`.githooks/`, via `core.hooksPath`), not in `.git/hooks`, which isn't versioned or shared.
- **Don't skip the hook to get a push through.** Fix the failing check. CI runs the same command, so skipping it only moves the failure.
- **Keep the hook tiny.** It calls `composer check:push` and nothing else, so CI, the hook and developers all run the same list.
- **Pin GitHub Actions to commit SHAs** in production repos and let Dependabot update them. A moved tag can run someone else's code with your repo's permissions.
- Set `persist-credentials: false` on checkout unless a later step needs to push.
