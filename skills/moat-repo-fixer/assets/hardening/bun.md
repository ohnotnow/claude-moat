# Bun supply-chain hardening (proactive — not a moat finding)

Proactive, opt-in hardening for **Bun** projects. moat does not flag any of this; re-running moat won't show it resolved — see `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, never a git mutation.

**Detect Bun — never on `package.json` alone** (it's shared with npm/pnpm/yarn):

- `bun.lock` (text/JSONC, the default since Bun **v1.2**) → Bun
- `bun.lockb` (legacy binary lockfile) → Bun, but **un-migrated** — recommend migrating: `bun install --save-text-lockfile --frozen-lockfile --lockfile-only`, then delete `bun.lockb` (also unlocks Dependabot, which doesn't support `bun.lockb`).
- `bunfig.toml` → Bun config present.
- A repo mid-migration can carry more than one lockfile — flag rather than assume.

**Bun is already safe-by-default on two of the axes**, so the action is *review/tune*, not *add*.

## Already-on: install-script allowlist (`trustedDependencies`)

Bun does **not** run lifecycle scripts (`postinstall` etc.) for arbitrary deps by default — it uses a built-in default-trusted list plus a `trustedDependencies` allowlist in `package.json`. This is the inverse of npm's opt-in `ignore-scripts`, and it's **on by default** — so the npm "add `ignore-scripts`" step is already done.

- The action is to **review `trustedDependencies`**: `bun pm untrusted` shows what was blocked; `bun pm default-trusted` shows the built-in list. Trust only what's needed, after review (`bun pm trust <pkg>`). **Discourage `bun pm trust --all`** (defeats the protection).
- `trustedDependencies` allows scripts for the listed package **only, not its transitive deps**.
- `ignoreScripts = true` in `bunfig.toml` is the hard global off-switch (overrides the allowlist) if a project wants zero lifecycle scripts. **What breaks:** packages that genuinely need a postinstall (native builds, `sharp`, `puppeteer`) — `bun pm untrusted` tells you which.

## Fix: release-age cooldown — native, in `bunfig.toml` (not `.npmrc`)

Bun **v1.3** added a native cooldown. It's a `bunfig.toml` key, **in seconds** — Bun does *not* read npm's `.npmrc` `min-release-age`.

```toml
[install]
minimumReleaseAge = 604800                  # 7 days, in seconds (259200 = 3 days)
minimumReleaseAgeExcludes = ["typescript"]  # trusted tooling exceptions, sparingly
```

Affects new resolution only; versions already in `bun.lock` are unchanged. *Known gaps to flag: `bunx --minimum-release-age` is reportedly a no-op, and some `bun update` paths don't apply it to transitive deps.* So treat it as protecting `bun install` / `bun add`.

## Fix: frozen installs (the `npm ci` axis)

`bun install --frozen-lockfile` fails if the lockfile would change (Bun auto-enables it when `CI` is set). Also `frozenLockfile = true` under `[install]` in `bunfig.toml`.

- **Dockerfile:** `RUN bun install --frozen-lockfile --production` after copying `package.json` + `bun.lock`. (Base-image / Bun-version pins → `pinned-versions.md`.) A `bun.lockb` repo should migrate to `bun.lock` first.

## Fix (Bun-only bonus): Security Scanner API

Bun **v1.3** added a Security Scanner API — set a scanner under `[install.security]` in `bunfig.toml` and Bun scans packages pre-write, cancelling installs on fatal findings:

```toml
[install.security]
scanner = "@socketsecurity/bun-security-scanner"
```

**What breaks / costs:** setting a scanner **disables auto-install**, adds a network/latency dependency, and a fatal finding hard-stops installs (incl. CI). Offer it; don't force it.

## Dependabot

Supports the `bun` ecosystem with `cooldown` — but **no security updates** for bun, and **no `bun.lockb`** (migrate to `bun.lock` first, Bun ≥ v1.1.39).

---
*Sources / last verified ~mid-2026: bun.com docs (lockfile, trusted deps, bunfig, npmrc, security-scanner-api); Bun v1.3 announcement (minimumReleaseAge, scanner); GitHub Dependabot supported-ecosystems & options. Flagged: the `.npmrc ignore-scripts` precedence caveat and the `bunx` / `bun update` cooldown gaps come from third-party reports / open issues — confirm current behaviour before asserting.*
