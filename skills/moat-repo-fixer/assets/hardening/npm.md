# npm supply-chain hardening (proactive — not a moat finding)

Proactive, opt-in hardening for **npm** projects. moat does not flag any of this, and re-running moat will not show it as resolved — see the orientation map and caveats in `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, one change at a time, never a git mutation.

**This guide is for npm specifically.** `package.json` alone doesn't prove npm — disambiguate on the lockfile first:

- `package-lock.json` → npm (this guide)
- `bun.lock` / `bun.lockb` / `bunfig.toml` → **Bun** → `bun.md`
- `pnpm-lock.yaml` / `pnpm-workspace.yaml` → **pnpm** (equivalents differ: pnpm uses `minimumReleaseAge` in **minutes**, default 1440 in pnpm 11, in `pnpm-workspace.yaml`) — don't write an npm `.npmrc`; tell the user the pnpm equivalent and stop.
- `yarn.lock` → **yarn** (its own mechanism) — likewise stop.

## Fix: `.npmrc` release-age + install-script lockdown

Two defensive settings:

```ini
# Don't install any version published less than a day ago. A poisoned release
# is usually detected and yanked within hours (chalk/debug ~2.5h, Shai-Hulud
# ~12h), so even a short cooldown filters most compromises. Value is in days.
min-release-age=1
# Don't run dependency lifecycle scripts — the postinstall hook most worms fire from.
ignore-scripts=true
```

**Write or merge — never clobber.** If `.npmrc` exists, `Read` it and add only the missing keys (preview as a `diff -u`); if it doesn't, create it (preview as a fenced block). Never overwrite registry/auth lines the user already has.

**`min-release-age` caveats — say these out loud before applying:**

- It needs **npm ≥ 11.10.0**. Older npm silently ignores the key, so a committed `.npmrc` gives false comfort. Recommend pinning the Node/npm version in CI so the setting actually bites.
- The value is in **days**. `1` is the floor that catches fast-yanked compromises; 3–7 buys more margin at the cost of lag on legitimate updates. It's a dial — mention it.

**Cooldown ≠ remediation — say this so the `.npmrc` isn't mistaken for a fix.** `min-release-age` and `ignore-scripts` defend against *future* poisoned releases; they do **nothing** for a version already in the tree that's *already* known-vulnerable (a stale `axios`, say). Clearing those is `npm audit` + an actual upgrade, not a cooldown. On a GitLab-only repo there's no Dependabot to raise that upgrade either (see the Dependabot note below), so flag the already-old deps explicitly rather than letting the new `.npmrc` imply they're handled.

**`ignore-scripts` is the blunt one — detect, then caution.** Before setting it, scan `package.json` (deps *and* devDeps) for packages that legitimately build on install, and tailor the warning:

- *Genuinely fragile* — `node-gyp`-based native modules, `node-sass`, anything with a known compile-on-install step → warn the install/build **will** break without an allowlist.
- *Degraded-but-works* — `esbuild` (and tools that bundle it: `vite`, `tsup`) still get their platform binary via `optionalDependencies`, so builds run, but the `.bin` shim stays on the slower JS path. Tailwind v4's `oxide` engine ships prebuilt platform packages and is unaffected. (`ignore-scripts` combined with `--no-optional` *does* break esbuild — never recommend that pairing.)
- For surgical control instead of all-or-nothing, point at `@lavamoat/allow-scripts`: it lets named packages keep their install scripts while blocking the rest.

After applying, recommend a smoke-test — `npm ci && npm run build` (or the repo's build script) — to confirm nothing downstream broke.

## Fix: `npm install` → `npm ci` in the Dockerfile

`npm ci` installs the **exact** locked versions, wipes `node_modules` first, and errors if `package.json` and the lockfile disagree — reproducible builds, no surprise re-resolution at image-build time.

**Find the install line with the `Grep`/`Read` tools, not `bash grep`.** A `bash grep 'npm install' Dockerfile` trips the user's edit-checker hook (and the dedicated file-search tools are the house default anyway) — use `Grep` for the `RUN npm install` pattern, `Read` to confirm the surrounding context, then `Edit`.

**Preconditions — check before suggesting the edit:**

1. A **`package-lock.json` must be committed** (and copied into the build stage — a `COPY package*.json` glob already includes it). `npm ci` fails outright without a lockfile. If there's none, *don't* just rewrite the line — recommend generating and committing one first (`npm install` once locally), then switching.
2. Only convert a **bare** `npm install` (optionally with `--production` / `--omit=dev`). Leave `npm install <pkg>` and `npm install -g <tool>` alone — `npm ci` takes no package arguments and will error.

Rewrite e.g. `RUN npm install && npm run build` → `RUN npm ci && npm run build`. Preview the Dockerfile change as a `diff -u`, confirm, apply. (For the base image and any pinned tool versions in that Dockerfile, see `pinned-versions.md`.)

**Don't copy the `.npmrc` cooldown into the Docker stage.** It's tempting to make the Docker build honour `min-release-age` too, but it's wrong: `min-release-age` gates version *resolution*, and `npm ci` installs already-locked versions — a cooldown there would *error* if a locked version were younger than the gate, breaking your own CI build of already-vetted deps. The cooldown belongs at the resolution layer (developers' `.npmrc` + Dependabot); `npm ci` simply reproduces the already-aged lockfile. Three layers, no conflict.

## The matching Dependabot cooldown

The automated-PR twin of `min-release-age` lives in `.github/dependabot.yml` (handled by the `repositories_have_dependabot_config` fix in `SKILL.md`). With `interval: weekly`, use `default-days: 7` rather than `1` — a 1-day cooldown rarely changes anything at a weekly cadence. Cooldown applies to version updates only, not security updates.

**GitHub-only:** Dependabot doesn't run on a GitLab-only repo, so this automated twin is inert there — see `SKILL.md` › *Supply-chain hardening* for the GitLab story (Dependency Scanning / Renovate, or periodic review against the dated pins).

---
*Sources / last verified ~mid-2026: npm `.npmrc` config docs (`min-release-age`, `ignore-scripts`); `@lavamoat/allow-scripts`; esbuild `optionalDependencies` behaviour. Re-check `min-release-age`'s minimum npm version before asserting it.*
