---
name: moat-repo-fixer
description: Apply in-repo security hardening fixes that Laravel's `moat` CLI flags for the current repository — pin GitHub Actions to commit SHAs, add a Dependabot config tailored to the repo's ecosystems, drop in a `SECURITY.md`, and restrict per-workflow token permissions. Also applies proactive npm supply-chain hardening beyond moat's own findings — an `.npmrc` release-age cooldown and install-script lockdown, a matching Dependabot cooldown, and `npm ci` in Dockerfiles. Use when the user is inside a repo and wants to action moat findings, or asks to "harden this repo", "fix moat issues here", "apply moat fixes", "harden npm", or "protect against npm supply-chain attacks". Settings-level fixes (branch protection, org defaults) are out of scope — those belong to the sibling `moat-org-fixer` skill.
---

# moat-repo-fixer

Walks through the in-repo fixes that moat (https://github.com/laravel/moat) flags. Strictly file-level changes inside the current repository. Settings-level fixes belong to `moat-org-fixer`.

## Choose scope first

Use `AskUserQuestion` up front to find out what the user wants:

- **A — this repo** — action moat's findings for the single repo we're in right now. This is the default when the skill is invoked from inside a repo. Proceed straight to *Source the findings* below.
- **B — all of an org's repos** — sweep every local checkout of an org's repos and fix them in turn. See *Mode B: every repo in an org* (near the end) for the orchestration; it reuses the per-repo flow once per repo.

Everything from *Source the findings* down to *Mode B* is the **per-repo flow** — used directly in Mode A, and run once per repo in Mode B.

## Source the findings (preference order)

1. **User-supplied JSON file** — if the user mentions a moat report (e.g. `moat.json`, `moat_report.json`), load it and filter `checks[].affected[]` for entries matching the current repo.

2. **Live `moat` run** — if `moat` is on `PATH`, detect the current repo's `owner/name` from `git remote get-url origin`, then run:
   ```
   moat --format json <owner>/<repo>
   ```
   **Always pass `--format json`** — without it `moat` writes ANSI-coloured terminal output that's painful to parse.

3. **Fallback** — if `moat` isn't installed, tell the user:
   > `moat` isn't installed. It's actively maintained against new attack vectors so it's the best source of what to worry about. Install from https://github.com/laravel/moat. I can fall back to a manual inspection (pin actions, check `.github/dependabot.yml`, `SECURITY.md`, workflow permissions) but I'll miss anything moat catches that I don't know to check for.

## Detecting the current repo

```
git remote get-url origin
```

Parse `owner/repo` from both forms:
- `https://github.com/owner/repo.git`
- `git@github.com:owner/repo.git`

## Filter findings

For each `checks[]` entry where `status == "fail"`, check whether the current repo appears in `affected[]`. A starting `jq` filter:

```
jq --arg repo "<repo>" '.checks[] | select(.status == "fail" and ((.affected // []) | any(.repo == $repo))) | {id, label, summary, why_enable, how_to_fix}' moat_report.json
```

Show the user the filtered list using moat's own `label` and `summary` — keeps the *reasons* current as moat evolves rather than baking them into this skill.

If no findings apply to this repo, say so and exit.

## Walk through each fix — one at a time

**Never bulk-apply.** Each fix: preview → confirm → apply. Then move on. **Never** run `git add`, `git commit`, `git push`, or any git mutation — the user's global instructions are explicit.

**What "preview" means in practice:**

- **Modifying an existing file** — use real `diff -u` against a preview copy:
  1. **Pre-flight sweep** at the start of the session — clear any leftover previews from an interrupted prior run:
     ```
     find . -name '.moat-preview-*' -type f -delete
     ```
  2. Make a preview copy next to the target. **In the project directory — don't use `/tmp`**; Claude Code's sandbox is tightening and writes outside the project root may be blocked in future.
     ```
     preview="$(dirname <original>)/.moat-preview-$(basename <original>)"
     cp <original> "$preview"
     ```
  3. **`Read` the preview, then `Edit` it.** Use `Edit` (not `Write`) — Claude Code's `Write` tool refuses to overwrite a file that hasn't been `Read` first, and `mktemp`+`Write` creates an awkward fail-then-Read-then-Write dance. `cp`+`Read`+`Edit` is one fewer step and plays to the tool's strengths (especially for many small changes in one file).
  4. Show the diff: `diff -u <original> "$preview"`
  5. **Confirmation style:** short previews (under ~30 lines) can use `AskUserQuestion`. Long previews — prefer a plain-text "Apply this change? (y/n)" in chat. `AskUserQuestion` overlays the diff so the user can't easily scroll, and `Ctrl+O` to expand the prompt dismisses it as declined.
  6. On confirmation, apply: `cp "$preview" <original> && rm "$preview"`
  7. On rejection, just `rm "$preview"`.

  Why this dance: developers read unified diffs fluently, and the diff matches what would actually land character-for-character. The leading `.` and `moat-preview-` prefix keep the file out of most globs and make any leftover visually obvious. In-project temp files future-proof against sandbox restrictions on `/tmp`.

- **Creating a new file** (e.g. `SECURITY.md`, `.github/dependabot.yml` where none exists) — show the proposed content in a fenced code block. No diff is meaningful when there's nothing to compare to.

- **Batching within a file** — if a single file has multiple changes (either of the same kind, *or across different fix categories*), produce one preview with *all* edits applied and show one `diff -u` for the lot. Don't make the user confirm 11 times for one workflow, and don't make them confirm the same workflow twice when both pinning and permissions touch it. The "one category at a time" rule is about ordering your *thinking* — not about producing multiple previews for the same file.

- Always clean up temp files whether the change was applied, rejected, or errored.

### Fix: `repositories_workflow_actions_are_pinned`

For every `.github/workflows/*.yml`:

1. Parse `uses:` entries pointing to external actions (`owner/repo@ref`).
2. Skip:
   - Refs that are already a full 40-char SHA
   - Local refs (`./.github/workflows/foo.yml`)
   - Docker (`docker://...`)
3. For each remaining ref, resolve to a full SHA:
   ```
   gh api repos/<owner>/<repo>/commits/<ref> --jq .sha
   ```
   **These calls are independent — fire them in parallel** (one tool message with N Bash calls) rather than serialising 12 round-trips.
4. **Check the action isn't stale or archived.** Same `gh api` call gives you the repo metadata:
   ```
   gh api repos/<owner>/<repo> --jq '{archived, pushed_at}'
   ```
   Flag (don't auto-skip) any action where `archived: true` or `pushed_at` is more than 2 years ago. These are real supply-chain smells: an abandoned action is one compromise away from being a problem. Surface them to the user with a recommendation to find a maintained alternative before pinning.

   **If you suggest a replacement, verify it first** with the same `archived`/`pushed_at` check — don't recommend something from training memory that may itself have gone stale since.
5. Rewrite as `uses: owner/repo@<full_sha> # <original_ref>` so a human reader can still see the version.
6. Preview the changes for the file (batch all refs in that file into one preview), confirm, apply.
7. If a ref is `@main`, `@master`, or a branch name, surface that as a smell — recommend the user pin to a tagged release first, then to its SHA.

### Fix: `repositories_have_dependabot_config`

1. Detect ecosystems present in the repo:
   - `composer.json` → `composer`
   - `package.json` → `npm`
   - `requirements.txt` / `pyproject.toml` → `pip`
   - `go.mod` → `gomod`
   - `Dockerfile` → `docker`
   - `.github/workflows/*.yml` → `github-actions` (almost always present, include unconditionally if `.github/workflows` exists)
2. Pick the closest starting template from `assets/dependabot/`:
   - `laravel.yml` — composer + npm + github-actions (full Laravel app)
   - `php-package.yml` — composer + github-actions (PHP library)
   - `base.yml` — github-actions only
3. Add/remove ecosystems to match what you actually detected.
4. **Include a `cooldown` per ecosystem.** This is the automated-PR twin of the `.npmrc` release-age cooldown (see *npm supply-chain hardening* below) — it stops Dependabot raising PRs for day-zero releases. The bundled templates already carry one:
   ```yaml
       cooldown:
         default-days: 7
   ```
   Why `7` here when the `.npmrc` uses `1`? The meaningful value is schedule-dependent: with `interval: weekly`, a 1-day cooldown rarely changes anything (most releases are already older than a day by the weekly run), whereas ~7 days actually dodges a freshly-poisoned release. Different layer, different sensible number — same goal. Cooldown applies to version updates only (not security updates) and accepts 1–90 days; tune `default-days` (and optionally the `semver-major-days` / `-minor-days` / `-patch-days` split) to taste.
5. Write to `.github/dependabot.yml`. Preview, confirm.
6. If `.github/dependabot.yml` already exists, diff the proposed against the existing and let the user decide what to merge.

### Fix: `repositories_have_security_policy`

**First, check for an org-wide default — before writing anything.** GitHub serves a default `SECURITY.md` to *every* repo in an org from a special `<owner>/.github` repository (it appears in each repo's Security tab automatically). Many developers — and moat itself — don't know this exists, so the real risk here is **drift**: adding a redundant per-repo file that overrides a perfectly good org default. `<owner>` comes from `git remote get-url origin`; check the canonical locations:

```
for p in SECURITY.md .github/SECURITY.md docs/SECURITY.md; do
  gh api "/repos/<owner>/.github/contents/$p" --jq .html_url 2>/dev/null && break
done
```

Belt-and-braces, this shows what GitHub *actually serves* to the current repo — a non-null URL on a repo moat flagged means an inherited default is already in play:

```
gh api /repos/<owner>/<repo>/community/profile --jq '.files.security_policy.html_url'
```

**If an org-wide default exists**, surface it instead of silently writing a file:

> This repo has no `SECURITY.md` of its own — **but** your org already provides a default via `<owner>/.github` (`<url>`), which already covers this repo's Security tab. Adding a repo-specific file would override that and start exactly the drift we want to avoid. moat flags this anyway because it only reads *this repo's* own contents (a known false-positive when an org default exists). Want to (a) leave it to the org default — recommended — or (b) add a repo-specific `SECURITY.md` because this repo genuinely needs different contact/policy details?

Only write a file if the user picks (b). Otherwise record it under "Skipped / deferred" with the reason ("covered by org-wide default at `<url>`") and move on.

**If there's no org default and you're hardening several repos**, mention the DRY alternative: a single `SECURITY.md` in a new `<owner>/.github` repo covers them all at once — that's `moat-org-fixer` territory and usually beats N per-repo files.

**To write a repo-specific file** (no org default, or the user chose (b)):

1. Get suggested name and email from the user's git config:
   ```
   git config user.name
   git config user.email
   ```
2. Use `AskUserQuestion` to offer these as defaults — the user can confirm with one click. Options:
   - "Use `<name>` / `<email>`" (the detected values, recommended)
   - "PVR only — no email contact" (skip the email line entirely)
   - "Other" — user types preferred values
3. Copy `assets/SECURITY.md` to the repo root and substitute `[YOUR NAME]` and `your@email.com`. If PVR-only was chosen, remove the entire "Email" bullet.
4. Preview, confirm, apply.

### Fix: `repositories_workflow_permissions_are_restricted`

**This finding is fixed only at the file level — there is no settings shortcut.** moat checks each workflow file for an explicit, minimal `permissions:` block. A common misconception (corrected after a real run) is that the org-wide default-token setting in `moat-org-fixer` clears this "for all repos at once" — it does **not**. That setting addresses a *different* finding (`repositories_actions_workflow_token_is_read_only`) and leaves this count untouched. Keep it as a complementary safety net, but the only thing that clears *this* finding is editing the files.

Add a top-level block to each `.github/workflows/*.yml`:

```yaml
permissions:
  contents: read
```

If a workflow legitimately needs more, keep the grant as narrow as possible and add a one-line comment explaining why.

**Common actions that need elevated permissions** (use this as a starting point — verify against the action's current docs if unsure, and prefer the narrowest grant):

| Action | Minimum permissions |
| --- | --- |
| `softprops/action-gh-release` | `contents: write` |
| `marvinpinto/action-automatic-releases` | `contents: write` |
| `peter-evans/create-pull-request` | `contents: write`, `pull-requests: write` |
| `actions/stale` | `issues: write`, `pull-requests: write` |
| `actions/labeler` | `pull-requests: write` |
| `release-drafter/release-drafter` | `contents: write`, `pull-requests: read` |
| `JamesIves/github-pages-deploy-action` | `contents: write` |
| Anything that pushes commits, tags, or releases | `contents: write` |
| Anything that comments on or labels PRs/issues | `pull-requests: write` and/or `issues: write` |

If you see an action not in the table, read its `action.yml` (`gh api repos/<owner>/<repo>/contents/action.yml --jq .content | base64 -d`) or its README to figure out what it needs — don't guess by granting `write: all`.

**Multi-job workflows:** set `permissions: contents: read` at the workflow root, and grant elevated permissions at the *job* level only on the jobs that need them. Don't grant the whole workflow `contents: write` just because one job cuts a release — the principle of least privilege applies per-job, not per-workflow.

Preview each workflow's changes (one preview per file, batching all the permissions blocks in that file — and combining with any pinning changes for the same file, per the batching rule in the guardrails), confirm, apply.

## Other possible findings

`moat` evolves. If the filtered findings include an `id` not handled above, **do not silently skip** — show the user moat's `label`, `description`, `why_enable`, and `how_to_fix` URL, and ask whether they want help with it. If the fix is a file change, work through it with the same preview-then-confirm pattern.

## npm supply-chain hardening (proactive — not a moat finding)

moat does **not** currently flag any of the things in this section. They're proactive hardening, prompted by the run of npm supply-chain attacks (the Sept 2025 `chalk`/`debug` compromise, the Shai-Hulud worm, the 2026 `axios` incident). Because moat didn't ask for them, they're **opt-in**: only offer them when the repo actually has the relevant files, and only proceed if the user says yes. Re-running moat afterwards will **not** show these as resolved — they were never findings (see *Re-verify* in the closing summary).

**When to offer — detect the files first:**
- `package.json` present → offer the `.npmrc` hardening (the Dependabot `cooldown` is handled in the dependabot fix above).
- `Dockerfile` / `Dockerfile.*` present and running `npm install` → offer the `npm ci` switch.
- Neither present → skip this section silently.

**Package-manager check.** This section is written for **npm** (`.npmrc`). If you find `pnpm-lock.yaml` / `pnpm-workspace.yaml` or `yarn.lock` instead, the equivalents differ — pnpm uses `minimumReleaseAge` in **minutes** (default 1440 in pnpm 11) in `pnpm-workspace.yaml`; yarn has its own mechanism. Don't write an npm `.npmrc` into a pnpm/yarn project — tell the user the equivalent and stop.

All the usual guardrails apply: preview → confirm → apply, one change at a time, never a git mutation.

### Proactive fix: `.npmrc` release-age + install-script lockdown

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

**`ignore-scripts` is the blunt one — detect, then caution.** Before setting it, scan `package.json` (deps *and* devDeps) for packages that legitimately build on install, and tailor the warning:
- *Genuinely fragile* — `node-gyp`-based native modules, `node-sass`, anything with a known compile-on-install step → warn the install/build **will** break without an allowlist.
- *Degraded-but-works* — `esbuild` (and tools that bundle it: `vite`, `tsup`) still get their platform binary via `optionalDependencies`, so builds run, but the `.bin` shim stays on the slower JS path. Tailwind v4's `oxide` engine ships prebuilt platform packages and is unaffected. (`ignore-scripts` combined with `--no-optional` *does* break esbuild — never recommend that pairing.)
- After applying, recommend a smoke-test — `npm ci && npm run build` (or the repo's build script) — to confirm nothing downstream broke.
- For surgical control instead of all-or-nothing, point at `@lavamoat/allow-scripts`: it lets named packages keep their install scripts while blocking the rest.

### Proactive fix: `npm install` → `npm ci` in the Dockerfile

`npm ci` installs the **exact** locked versions, wipes `node_modules` first, and errors if `package.json` and the lockfile disagree — reproducible builds, no surprise re-resolution at image-build time.

**Preconditions — check before suggesting the edit:**
1. A **`package-lock.json` must be committed** (and copied into the build stage — a `COPY package*.json` glob already includes it). `npm ci` fails outright without a lockfile. If there's none, *don't* just rewrite the line — recommend generating and committing one first (`npm install` once locally), then switching.
2. Only convert a **bare** `npm install` (optionally with `--production` / `--omit=dev`). Leave `npm install <pkg>` and `npm install -g <tool>` alone — `npm ci` takes no package arguments and will error.

Rewrite e.g. `RUN npm install && npm run build` → `RUN npm ci && npm run build`. Preview the Dockerfile change as a `diff -u`, confirm, apply.

**Don't copy the `.npmrc` cooldown into the Docker stage.** It's tempting to make the Docker build honour `min-release-age` too, but it's wrong: `min-release-age` gates version *resolution*, and `npm ci` installs already-locked versions — a cooldown there would *error* if a locked version were younger than the gate, breaking your own CI build of already-vetted deps. The cooldown belongs at the resolution layer (developers' `.npmrc` + Dependabot); `npm ci` simply reproduces the already-aged lockfile. Three layers, no conflict.

## Mode B: every repo in an org

This sweeps every *local checkout* of an org's repos and applies the per-repo flow to each. It exists because local directory names — and even remote names — drift from the GitHub repo name (a repo's GitHub remote might be `origin`, or `github` when the primary is an on-prem GitLab and GitHub is a backup), so checkouts can't be found by folder name alone.

**1. Gather inputs.** Ask the user for:
- the **org** name (e.g. `UoGSoE`)
- the **base path** that holds their project folders (e.g. `~/Documents/code`)

**2. Build the repo → local-path map.** Run the bundled resolver — it reads *every* remote of every immediate sub-directory and matches on the github.com URL, not the folder name. It lives under this skill's base directory (shown when the skill loads); invoke via `bash` so it works regardless of the exec bit:
```
bash <skill-dir>/assets/map_repos_to_local <org> <base-path>
```
It prints CSV `repo,local_path,current_branch,matched_remote,remote_url` to stdout, plus a counts line and any duplicate-repo warnings to stderr. Capture stdout.

**3. Source the org's findings** (same preference order as Mode A, at org scope): if the user has a recent moat **report**, use it; otherwise offer to run `moat --format json <org>`. Keep only the **file-level** findings this skill owns — `repositories_workflow_actions_are_pinned`, `repositories_workflow_permissions_are_restricted`, `repositories_have_security_policy`, `repositories_have_dependabot_config`. Settings-level findings belong to `moat-org-fixer`: name them in the summary but don't act on them here.

**4. Join findings to checkouts.** For each flagged repo, look it up in the map and bucket it:
- **one local checkout** → queue for fixing
- **no local checkout** → "clone needed" (list at the end; never clone automatically)
- **more than one checkout** (the dupe warning) → show the candidate paths and **ask the user which one** before queuing

**5. Fix each queued repo, one at a time.** For each, in turn:
- `cd` into its `local_path`.
- **Check the working tree first.** Show the repo's `current_branch` and `git status --porcelain`. If it's on a non-default branch or has uncommitted changes, **surface that and ask** whether to (a) fix here anyway, (b) skip this repo, or (c) stop the sweep. Never edit a dirty or feature-branch checkout silently.
- Run the **per-repo flow above** for that repo, scoping the findings to that repo's entries (you already have them from step 3 — don't re-run moat per repo). The `SECURITY.md` org-default pre-check still applies — and in an org sweep there's very likely an org default, so expect to recommend skipping the per-repo file.
- The proactive **npm supply-chain hardening** section is part of the per-repo flow too, but it's opt-in and non-moat. In an org sweep, ask **once** whether to apply it to all qualifying repos (those with a `package.json` / `Dockerfile`), then apply consistently — don't re-prompt for every repo.
- Honour every guardrail: one fix at a time, preview → confirm → apply, never a git mutation.

**6. Closing summary (org sweep).** On top of the per-repo summaries, give an org-level roll-up:
- **Fixed** — repo → which files changed.
- **Skipped** — repo + reason (dirty tree, on a feature branch, user declined).
- **Clone needed** — flagged repos with no local checkout.
- **Ambiguous** — repos with multiple checkouts, and which path you used.
- **Settings-level (for `moat-org-fixer`)** — the org/settings findings you deliberately left alone.
- **Commit/push is the user's job** — an org sweep leaves uncommitted changes across many working trees. Remind them clearly; you never run git.

## Guardrails

- Never run git mutations (`add`, `commit`, `push`, `checkout`, `reset`, `restore`, etc.).
- Never bulk-apply across fix categories. One category at a time, one file at a time across categories.
- In **Mode B** (org sweep), also work one *repo* at a time — finish or skip a repo before moving to the next; never fan out edits across repos.
- **Within a single file**, batch multiple changes of the same kind into one preview and apply in one pass. Don't make the user confirm 11 nearly-identical line edits in one workflow.
- Always show a preview before writing — `diff -u` against a temp file for modifications, a fenced code block for new files.
- Already-correct items get skipped silently — don't rewrite a workflow file just to touch it.
- If you can't resolve something (e.g. a private action you don't have `gh api` access to), ask the user — don't guess a SHA.
- The **npm supply-chain hardening** section is proactive, not a moat finding — only offer it when the relevant files (`package.json` / `Dockerfile`) exist, and only act on the user's say-so. Never write an npm `.npmrc` into a pnpm/yarn project.

## After all fixes

Print a structured summary with **all** of the following sections — don't skip any, even when empty. (In **Mode B**, this is the per-repo summary; Mode B's step 6 rolls these up across the org.)

1. **Files changed** — list each path that was written, with a one-line note on what changed.
2. **Skipped / deferred** — anything you didn't do and why (e.g. "action X is archived, recommended replacing before pinning").
3. **Out-of-scope findings (for `moat-org-fixer`)** — explicitly list every moat finding for this repo whose fix is a setting rather than a file. The user needs to see the *remaining* work, not just what was done. For each: the finding's `label` and the matching repo entry from `affected[]`.
4. **Next steps**:
   - The user needs to commit and push these changes themselves (you don't do git).
   - The org-level default-token fix in `moat-org-fixer` cascades to all repos and clears `repositories_actions_workflow_token_is_read_only` in one go — recommend it as a complementary safety net. But it does **not** clear `repositories_workflow_permissions_are_restricted` (the per-workflow `permissions:` block we fixed here): the two are different findings, so don't suggest deferring the file fix in favour of the org setting.
5. **Re-verify (optional)** — offer to re-run `moat --format json <owner>/<repo>` after the user has pushed. Be honest about the caveat: file-level checks (`workflow_actions_are_pinned`, `workflow_permissions_are_restricted`, `have_security_policy`, `have_dependabot_config`) should flip locally after push; anything that reads GitHub-side state (branch protection, org policy) won't change until the matching settings are also updated. The proactive npm hardening (`.npmrc`, Dockerfile `npm ci`, Dependabot `cooldown`) won't appear in moat output at all — moat doesn't check for these, so don't expect any count to drop; the protection is real, just invisible to moat.
