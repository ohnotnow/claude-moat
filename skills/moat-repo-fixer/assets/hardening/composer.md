# Composer / PHP supply-chain hardening (proactive — not a moat finding)

Proactive, opt-in hardening for **Composer** projects. moat does not flag any of this; re-running moat won't show it resolved — see `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, one change at a time, never a git mutation. Trigger on `composer.json`.

**Composer's threat model differs from npm's — and that changes which control matters most.** Verbatim from the docs: *"Only scripts defined in the root package's `composer.json` are executed. If a dependency … specifies its own scripts, Composer does not execute those."* So a dependency's install scripts **don't auto-run** — the npm `ignore-scripts` analogue is *secondary* here. The real dependency-side code-execution surface is **plugins**, which *do* auto-load. So the priority control is `allow-plugins`, not `--no-scripts`.

## Fix (primary): `allow-plugins` deny-by-default allowlist

`config.allow-plugins` (Composer **2.2.0+**) restricts which packages may execute plugin code during a Composer run. Default is `{}` — deny-by-default. In non-interactive mode (CI/Docker) Composer **fails** if an unlisted plugin is present, so it fails closed.

```json
{
  "config": {
    "allow-plugins": {
      "composer/installers": true,
      "phpstan/extension-installer": true
    }
  }
}
```

- **Read `composer.json` first**, merge into any existing `config` block — don't clobber. Preview as a `diff -u`.
- List only the plugins the project actually uses; **never set `"allow-plugins": true`** (the wildcard re-opens the hole).
- **What breaks:** a newly added dependency that ships a plugin will refuse to install in CI until it's added to the list — that's the feature working. Tell the user.

## Fix: locked installs — Composer is already `npm ci`

`composer install` with a committed `composer.lock` installs the **exact** pinned versions; it only resolves fresh via `composer update`. So there's no command swap to make — the hardening is ensuring CI/Docker run `install`, never `update`.

- Composer only *warns* on lock drift (unlike `npm ci`, which errors). For a hard CI gate that the lock matches `composer.json`, add `composer validate --no-check-publish` before install.
- **Dockerfile (production image):**
  ```dockerfile
  RUN composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader
  ```
  `--no-dev` drops dev tooling; `--no-interaction` makes the `allow-plugins` gate fail closed. (Base-image / Composer-version pins in that Dockerfile → `pinned-versions.md`.)
- `--no-scripts` is available as defence-in-depth but is **secondary** (dep scripts don't auto-run). In a Laravel build it can break package discovery / autoload-dump — only add it if the build genuinely needs no root scripts, or run `php artisan package:discover` explicitly afterwards.

## Fix: auditing in CI

`composer audit` (Composer **2.4+**) checks the locked set against the Packagist advisory API (GitHub Advisory DB + `FriendsOfPHP/security-advisories`). Since **2.7** it also fails on abandoned packages; since **2.10** it surfaces and fails on malware by default, and the `config.policy` framework (advisories / abandoned / malware) runs even on a plain `composer install`. Recommend a `composer audit` CI step (`--no-dev` to scope to production deps).

## Release-age cooldown

**Composer has no native release-age cooldown yet** — `minimum-release-age` is reserved in the policy schema but blocked on Packagist making release metadata immutable first (composer/composer #12877). Watch this space; don't tell the user Composer has it today.

Until then, the cooldown comes from **Dependabot** on the `composer` ecosystem (`.github/dependabot.yml`, handled in `SKILL.md`) — supports `default-days` and the per-semver split. Version updates only, not security updates.

## TLS / HTTPS posture — verify, don't change

`secure-http` (default `true`) and `disable-tls` (default `false`) are already safe by default. The action is to *confirm* they haven't been flipped off in `composer.json` or via env (`COMPOSER_DISABLE_TLS`) — not to set them.

---
*Sources / last verified ~mid-2026: getcomposer.org config / scripts / plugins / CLI docs; Packagist supply-chain blog (min-release-age status); Composer 2.10 malware policy (laravel-news). Confirm `composer audit` exit-code semantics and the native-cooldown status before asserting them.*
