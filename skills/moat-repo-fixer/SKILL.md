---
name: moat-repo-fixer
description: Apply in-repo security hardening fixes that Laravel's `moat` CLI flags for the current repository — pin GitHub Actions to commit SHAs, add a Dependabot config tailored to the repo's ecosystems, drop in a `SECURITY.md`, and restrict per-workflow token permissions. Also applies proactive supply-chain hardening beyond moat's own findings, tailored per ecosystem — npm/Bun install-script and release-age cooldowns, Composer `allow-plugins`, Python (pip/uv) wheels-only installs and hash-pinning, Go checksum and `govulncheck` checks, plus pinning Docker image tags to digests and catching stale pinned versions across Dockerfiles, compose/Swarm stack files and CI config. Use when the user is inside a repo and wants to action moat findings, or asks to "harden this repo", "fix moat issues here", "apply moat fixes", "harden npm/composer/python/go/bun", or "protect against supply-chain attacks". Can optionally deliver the applied fixes as a GitLab merge request via `glab` when the repo's primary remote is an on-prem/self-managed GitLab (detected from the git remotes, or when the user mentions GitLab) — e.g. "raise a merge request with the moat fixes", "open a gitlab MR", "push these to our gitlab". Settings-level fixes (branch protection, org defaults) are out of scope — those belong to the sibling `moat-org-fixer` skill.
---

# moat-repo-fixer

Walks through the in-repo fixes that moat (https://github.com/laravel/moat) flags. Strictly file-level changes inside the current repository. Settings-level fixes belong to `moat-org-fixer`.

## Check for prior org context first

Before asking the user anything, **if a cross-project memory tool is available (e.g. the `user-memories` MCP), search it for prior moat/hardening context for this org** — the hosting topology (which remote is primary, GitHub vs GitLab), any settings-level findings that are *deliberate accepted-risk decisions* (so you don't re-propose "fixing" what the team consciously chose to leave), and any "standard recipe" captured from earlier repos (what's usually already hardened, the recurring file-level work, ecosystem quirks). Foregrounding these avoids re-asking settled questions and re-investigating the same facts on every repo in a batch. Treat recalled memories as point-in-time — confirm against the live repo before acting on them.

## Choose scope first

Use `AskUserQuestion` up front to find out what the user wants:

- **A — this repo** — action moat's findings for the single repo we're in right now. This is the default when the skill is invoked from inside a repo. Proceed straight to *Source the findings* below.
- **B — all of an org's repos** — sweep every local checkout of an org's repos and fix them in turn. See *Mode B: every repo in an org* (near the end) for the orchestration; it reuses the per-repo flow once per repo.

Everything from *Source the findings* down to *Mode B* is the **per-repo flow** — used directly in Mode A, and run once per repo in Mode B.

## Source the findings (preference order)

**First, bucket the repo's remotes** (jump down to *Detecting the current repo*, then come back) — that decides whether moat even applies:

- **No `github.com` remote at all** → moat has nothing to scan; it only reads GitHub-side state. **Skip this entire findings flow** and go straight to *Supply-chain hardening* (proactive, host-agnostic) and, if the user wants it, *Delivery*. Don't run `moat` against a repo with no GitHub identity — it has no target. **If such a repo still carries a `.github/workflows/` directory**, don't silently ignore it: those workflows can't run without a GitHub home, so they're most likely vestigial. Surface them and offer to (a) remove them as cruft, or (b) — *only if the user push-mirrors to GitHub* — pin them proactively (the pinning/permissions fixes below, run by hand rather than off a moat finding). Ask which; don't assume either way.
- **A `github.com` remote exists** → continue with the preference order below.

1. **User-supplied JSON file** — if the user mentions a moat report (e.g. `moat.json`, `moat_report.json`), load it and filter `checks[].affected[]` for entries matching the current repo.

2. **Live `moat` run** — if `moat` is on `PATH`, detect the current repo's GitHub `owner/name` (from the GitHub remote — see *Detecting the current repo*; it isn't always `origin`), then run:
   ```
   moat --format json <owner>/<repo>
   ```
   **Always pass `--format json`** — without it `moat` writes ANSI-coloured terminal output that's painful to parse.

3. **Fallback** — if `moat` isn't installed, tell the user:
   > `moat` isn't installed. It's actively maintained against new attack vectors so it's the best source of what to worry about. Install from https://github.com/laravel/moat. I can fall back to a manual inspection (pin actions, check `.github/dependabot.yml`, `SECURITY.md`, workflow permissions) but I'll miss anything moat catches that I don't know to check for.

## Detecting the current repo

moat reads GitHub-side state, so for **finding** purposes you need the repo's GitHub `owner/repo`. Parse it from the GitHub remote (whatever it's *named* — see below), from both forms:
- `https://github.com/owner/repo.git`
- `git@github.com:owner/repo.git`

**Enumerate *all* remotes, not just `origin`, and key off the *host*, not the remote's name.** Naming conventions vary and even invert: a repo backed by an on-prem/self-managed GitLab might have `origin` = GitLab and GitHub a backup named `github` — or the exact opposite (`origin` = GitHub, with the GitLab remote named `gitlab`). Don't assume from the name; bucket by host. The full remote list tells you both the GitHub identity (for moat) *and* how to **deliver** the fixes later (see *Delivery*).

```
git remote -v
```

Bucket the remotes by host and decide the **primary remote** — this drives the *Delivery* step and the closing *Next steps*:
- **`github.com` remote(s) only** → the existing GitHub flow; no MR step.
- **A non-GitHub (`gitlab.com` / on-prem / custom) remote only** → the GitLab delivery path.
- **Both** → `AskUserQuestion`: *"which remote is your true primary?"* The answer decides whether the GitLab MR step runs and how *Next steps* is worded.
- If the user explicitly mentions GitLab, bias to the GitLab path regardless.

(This is the no-GitHub case the top of *Source the findings* routes on: no moat findings — moat only scans GitHub — but the proactive *Supply-chain hardening* still applies and can still be delivered as a GitLab MR, and a dormant `.github/workflows/` gets surfaced rather than silently ignored.)

(If the user has a GitLab remote - you can call on the `glab` skill for guidences with the latest version of the gitlab cli)

## Filter findings

For each `checks[]` entry where `status == "fail"`, check whether the current repo appears in `affected[]`. A starting `jq` filter:

```
jq --arg repo "<repo>" '.checks[] | select(.status == "fail" and ((.affected // []) | any(.repo == $repo))) | {id, label, summary, why_enable, how_to_fix}' moat_report.json
```

Show the user the filtered list using moat's own `label` and `summary` — keeps the *reasons* current as moat evolves rather than baking them into this skill.

If no findings apply to this repo, say so and exit.

## Reconcile the scanned branch with your checkout

**moat scans a branch — and it may not be the one you have checked out.** moat evaluates GitHub-side state for the repo's **default branch**, which can differ from your local checkout (e.g. a repo created with `gh repo create` while on a feature branch keeps *that* branch as the GitHub default). Each finding carries the branch it came from in `affected[].branch`. If you fix the wrong branch's files, your edits won't match what moat flagged, the two branches may even hold *different* workflow files, and a re-run stays red no matter how good your fixes are.

Before editing anything, reconcile three things:

```
gh api repos/<owner>/<repo> --jq .default_branch   # the branch moat scanned
git branch --show-current                           # what you have checked out
```
…plus the `affected[].branch` on the findings themselves.

- **All three agree** → proceed normally.
- **They differ** → **stop and surface it. Don't silently edit the local branch.** Explain the mismatch and recommend one of:
  - reset the GitHub **default branch** to the branch you actually ship from (a repo-settings change — often the real fix when the default was set by accident), then re-run moat; or
  - check out the branch moat scanned, so your edits land where the findings are.

  **Never switch branches yourself** — `git checkout` is a git mutation and the user owns it. Hand them the choice and the commands, and wait for them to resolve it before walking the fixes.

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

### Resolution cache (optional — for multi-repo sweeps)

Across a dozen repos the same identifiers recur constantly — `actions/checkout@v4`, `docker/build-push-action@v6`, `node:22`. Re-resolving each per repo burns time *and* tokens: every `gh api`/registry result lands in context and is re-billed on each later turn, so overlapping refs are paid for over and over. An optional on-disk cache lets a sweep resolve each unique identifier **once**. It covers both this section's action SHAs/metadata and the image digests in `pinned-versions.md`.

- **Single-repo (Mode A) with no prior cache has nothing to *reuse* — but still *write* what you resolve (write-through).** The cache earns its keep from the *second* repo on (a Mode B sweep, or repeated Mode A runs over a day or two) — but that only works if the first run *seeded* it. Skipping the write entirely on a lone run defeats the "repeated Mode A" benefit, so seed it anyway unless you're certain this is a genuine one-off with no follow-ups.
- **Location:** `~/.cache/moat-repo-fixer/pins.tsv`. **Test writability first** — `mkdir -p ~/.cache/moat-repo-fixer 2>/dev/null && [ -w ~/.cache/moat-repo-fixer ]`; if it fails, **silently skip caching and resolve live**. The cache is an optimization, never a correctness dependency, and writes outside the project root may be sandbox-blocked.
- **Format:** one tab-separated entry per line, with a *per-entry* timestamp so the TTL works granularly:
  ```
  action-sha     actions/checkout@v4     <40-char-sha>                       2026-06-08T14:03:00Z
  action-meta    actions/checkout        archived=false;pushed=2026-05-30    2026-06-08T14:03:00Z
  image-digest   node:22                 sha256:<…>                          2026-06-08T14:03:00Z
  ```
- **TTL: 48h (a sweep), not weeks.** Tags move (`@v4`, `node:22` are republished in place) and even concrete tags get rebuilt (the `redis:8.2.2` OS-patch case). A long TTL pins everyone to stale, unpatched digests — the very "pinned-and-forgotten → reproducibly vulnerable" trap this skill warns against. On a hit newer than the TTL, reuse; otherwise re-resolve and rewrite the entry.
- **If you ever raise the TTL, carve `action-meta` out of the cache.** The archived/`pushed_at` check is a *security signal*, not a freshness nicety — it's the one answer you least want stale (a recently-archived action slipping through). 48h is narrow enough to cache it; weeks are not.
- **The cache feeds the *preview*, never the apply.** The human still sees the actual SHA/digest in the `diff -u` and confirms, so a stale or wrong cache line can't land silently.
- **Bonus:** within the TTL every repo pins a recurring ref to the *same* SHA — uniform across the org, and one Dependabot bump moves the whole fleet.

### Fix: `repositories_workflow_actions_are_sha_pinned`

(moat renamed this from `repositories_workflow_actions_are_pinned` — match on the current id; older reports may carry the old name.)

For every `.github/workflows/*.yml`:

1. Parse `uses:` entries pointing to external actions (`owner/repo@ref`).
2. Skip:
   - Refs that are already a full 40-char SHA
   - Local refs (`./.github/workflows/foo.yml`)
   - Docker (`docker://...`)
3. For each remaining ref, resolve to a full SHA — **check the resolution cache first** (see *Resolution cache* above); only resolve live on a miss or stale entry:
   ```
   gh api repos/<owner>/<repo>/commits/<ref> --jq .sha
   ```
   **The live calls are independent — fire them in parallel** (one tool message with N Bash calls) rather than serialising 12 round-trips. Write each freshly-resolved SHA back to the cache.
4. **Check the action isn't stale or archived** (cache key `action-meta` — one call per *action repo*, not per ref, so it dedups hard across a sweep). The repo-metadata call:
   ```
   gh api repos/<owner>/<repo> --jq '{archived, pushed_at}'
   ```
   Flag (don't auto-skip) any action where `archived: true` or `pushed_at` is more than 2 years ago. These are real supply-chain smells: an abandoned action is one compromise away from being a problem. Surface them to the user with a recommendation to find a maintained alternative before pinning.

   **If you suggest a replacement, verify it first** with the same `archived`/`pushed_at` check — don't recommend something from training memory that may itself have gone stale since.

   **Prefer a replacement the repo already trusts.** Before reaching for an unfamiliar action, check whether another workflow in the *same repo* already uses a maintained action for the same job — recommend consolidating onto that one rather than introducing a new dependency. (Real example: a `main.yml` pinning the archived `marvinpinto/action-automatic-releases@latest` while a sibling `cli-release.yml` already uses the maintained `softprops/action-gh-release` — the fix is to switch to what's already in the repo.) Still verify it with the `archived`/`pushed_at` check.

   **This check is branch-scoped — "same repo" really means "same branch".** The sibling workflow you'd consolidate onto must exist *on the branch moat scanned / the one you're editing* (see *Reconcile the scanned branch*). The `cli-release.yml`/`softprops` example only helps if that file is on the branch you're fixing — when the finding sits on an accidental default like `feature/save-form` and the maintained alternative only lives on `master`, the swap isn't available on the branch that's actually red. Confirm the candidate exists on the right branch before recommending it.
5. Rewrite as `uses: owner/repo@<full_sha> # <original_ref>` so a human reader can still see the version.
6. Preview the changes for the file (batch all refs in that file into one preview), confirm, apply.
7. If a ref is `@main`, `@master`, a branch name, **or a moving tag like `@latest` / `@stable`**, surface that as a smell — these are republished in place, so they're as mutable as a branch. Recommend pinning to a specific tagged release first, then to its SHA. (`@latest` is common on release actions and especially worth catching.)

> **Settings-level partner (for `moat-org-fixer`):** pinning the refs by hand is only half of it. moat also reports `repositories_enforce_workflow_actions_sha_pinning` — the repo/org **setting** that *requires* pinned SHAs, so an unpinned ref can't be reintroduced later. That's a settings fix (org-fixer's job), not a file edit, but name it in your out-of-scope summary so the user sees the pair.

### Fix: `repositories_have_dependabot_config`

**What moat actually checks here is narrow:** it passes as long as the `github-actions` ecosystem is tracked in `.github/dependabot.yml`. So if a config already lists `github-actions`, this finding is **already green** — and *adding* the other ecosystems below (npm, composer, docker, gomod, …) is **proactive completeness**, not closing a moat finding. Like the supply-chain section, those extra entries won't move any moat count. Worth doing, but be honest with the user about which is closing a finding and which isn't.

1. Detect ecosystems present in the repo:
   - `composer.json` → `composer`
   - `package.json` → `npm`
   - `requirements.txt` / `pyproject.toml` → `pip`
   - `go.mod` → `gomod`
   - `Dockerfile` → `docker` — also raises base-image update PRs, the ongoing twin of the one-off `pinned-versions.md` check (see *Supply-chain hardening*)
   - `.github/workflows/*.yml` → `github-actions` (almost always present, include unconditionally if `.github/workflows` exists)

   **Look in subdirectories, not just the repo root.** A monorepo buries ecosystems one level down — a Go CLI at `cli/go.mod` inside a Laravel app, a frontend at `web/package.json`. Detect marker files wherever they live (e.g. `find . -name go.mod -not -path '*/vendor/*' -not -path '*/node_modules/*'`) and set each Dependabot entry's `directory:` to the **directory of the marker** (`/cli`, not `/`). Root-only detection silently misses these — and they're often the components that ship binaries, so they matter most.
2. Pick the closest starting template from `assets/dependabot/`:
   - `laravel.yml` — composer + npm + github-actions (full Laravel app)
   - `php-package.yml` — composer + github-actions (PHP library)
   - `base.yml` — github-actions only
3. Add/remove ecosystems to match what you actually detected.
4. **Include a `cooldown` per ecosystem.** This is the automated-PR twin of the per-ecosystem release-age cooldown (see *Supply-chain hardening* below) — it stops Dependabot raising PRs for day-zero releases. The bundled templates already carry one:
   ```yaml
       cooldown:
         default-days: 7
   ```
   Why `7` here when the `.npmrc` uses `1`? The meaningful value is schedule-dependent: with `interval: weekly`, a 1-day cooldown rarely changes anything (most releases are already older than a day by the weekly run), whereas ~7 days actually dodges a freshly-poisoned release. Different layer, different sensible number — same goal. Cooldown applies to version updates only (not security updates) and accepts 1–90 days; tune `default-days` (and optionally the `semver-major-days` / `-minor-days` / `-patch-days` split) to taste.
5. Write to `.github/dependabot.yml`. Preview, confirm.
6. If `.github/dependabot.yml` already exists, diff the proposed against the existing and let the user decide what to merge.

**Check existence robustly, and cross-check GitHub when moat disagrees with you.** Test the path on its own (`test -f .github/dependabot.yml`) rather than folding it into a combined `ls a b c 2>/dev/null || echo none` — a combined `ls` exits non-zero if *any* listed file is missing, so the `|| echo none` fires even when the file is present and you can end up believing it's absent when it isn't. And if moat **passes** `repositories_have_dependabot_config` while you think there's no local file (or vice-versa), don't carry the contradiction: that's a tell your local tree and GitHub's scanned branch have diverged. Cross-check what GitHub actually serves — the same habit the `SECURITY.md` fix uses below:

```
gh api repos/<owner>/<repo>/contents/.github/dependabot.yml --jq .name
```

then reconcile (see *Reconcile the scanned branch*) before editing.

### Fix: `repositories_have_security_policy`

**Current moat usually *passes* this when an org-wide default exists — so you may not see it as a finding at all.** GitHub serves a default `SECURITY.md` to *every* repo in an org from a special `<owner>/.github` repository (it appears in each repo's Security tab automatically), and moat now reads that inherited default and counts it as satisfied (fixed via [laravel/moat#18](https://github.com/laravel/moat/issues/18), ~June 2026 — *older* moat versions false-positived here, so a pre-fix moat may still flag it).

So the real risk is **drift**: adding a redundant per-repo file that overrides a perfectly good org default. **Check for an org-wide default before writing anything** — it matters most on the *manual fallback path* (moat not installed, so you can't lean on its now-correct pass) and as plain DRY hygiene. `<owner>` is the GitHub owner (from the GitHub remote — see *Detecting the current repo*; not necessarily `origin`); check the canonical locations:

```
for p in SECURITY.md .github/SECURITY.md docs/SECURITY.md; do
  gh api "/repos/<owner>/.github/contents/$p" --jq .html_url 2>/dev/null && break
done
```

Belt-and-braces, this shows what GitHub *actually serves* to the current repo — a non-null URL on a repo moat flagged means an inherited default is already in play:

```
gh api /repos/<owner>/<repo>/community/profile --jq '.files.security_policy.html_url'
```

**Trust the direct `<owner>/.github` contents check over this one.** `community/profile` is the *less* reliable of the two — it has returned `null` for `security_policy.html_url` on a repo that demonstrably had an org default (the endpoint can lag, or miss a non-standard default branch / private repo). So a non-null profile URL is useful *confirmation*, but a `null` here does **not** mean there's no org default — defer to the direct contents check above before concluding the repo is uncovered.

**If an org-wide default exists**, surface it instead of silently writing a file:

> This repo has no `SECURITY.md` of its own — **but** your org already provides a default via `<owner>/.github` (`<url>`), which already covers this repo's Security tab. Adding a repo-specific file would override that and start exactly the drift we want to avoid. (Current moat already counts the inherited default as a pass, so you're most likely seeing this on the manual fallback path, or on an older moat.) Want to (a) leave it to the org default — recommended — or (b) add a repo-specific `SECURITY.md` because this repo genuinely needs different contact/policy details?

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

**Before defaulting a workflow to `contents: read`, read what its jobs actually do — don't set the floor reflexively.** A *monolithic single-job* workflow that builds **and** cuts a release (one job running tests, then `softprops/action-gh-release` or `marvinpinto/action-automatic-releases`) genuinely needs `contents: write`; dropping it to `read` will break the release. The elevated-permissions table below tells you what such a job needs — but only if you've read the steps first. Splitting that one job into separate build and release jobs *would* let you scope `write` to just the release job (true least-privilege), but that's a refactor of someone's pipeline — flag it as an option, don't silently restructure it.

**Read `repositories_actions_workflow_token_is_read_only` alongside this, and settle the release-action decision *first*.** Whether the workflow needs `write` at all is *driven* by what happens to the release step, so decide that before the `permissions:` block. Two coupled signals:

- If moat **passes** `repositories_actions_workflow_token_is_read_only`, the default token is already read-only — so a release step in the workflow likely *couldn't write anyway*, a tell that it's broken or unused. (It's a *different* finding from this one — see the note at the top of this section — but it's the cleanest evidence of whether the release step is load-bearing.)
- The release-step choice is pin / replace / **drop**. Keeping or pinning it still needs `contents: write`; replacing it needs write for the new action too; **dropping** it makes the whole workflow trivially `contents: read`. So settle that choice and the permissions floor falls out of it.

**Short-circuit for a fallback release step.** Walking the full reasoning every time is tedious when the answer is always the same — common where **GitHub Actions is a secondary/fallback CI and the project's real CI/release runs elsewhere** (e.g. a GitLab-primary repo where GitHub Actions is only a backup — you'll have spotted this from the remotes in *Detecting the current repo*). When you see a monolithic build+release job whose **only** write-needing step is the release, offer a *single consolidated question* instead of the multi-turn dance — but keep it a question, and state the consequence inline (dropping a release step is destructive for a project whose GitHub release *is* how it ships):

> This workflow's only step needing `contents: write` is the release step (`<action>`). I can **drop the release step and set the workflow to `contents: read`** in one go — tagged builds still build/push images, they just won't create a GitHub *release* entry any more. Fine if GitHub Actions is a fallback and your real release runs elsewhere; **not** fine if the GitHub release is how you distribute. Options: **(a) drop it → `contents: read`**, (b) keep the release and scope `write` to just that job, (c) keep as-is. Which?

**Lead with (a) only when the fallback signal is present** (a non-GitHub primary remote, or the user has said GitHub Actions is a backup); otherwise present the three options even-handedly and let the user choose. In **Mode B** (org sweep) this is a natural *ask-once*: confirm the policy for the whole sweep — *"for any repo whose only write-needing step is a fallback release step, drop it and go read-only?"* — and apply it consistently rather than re-prompting per repo.

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

**An *existing* `permissions:` block can still be too broad — tighten it even if moat is quiet.** moat may treat a workflow as satisfied simply because *a* block is present, so a workflow granting `contents: write` at the root of a multi-job pipeline can pass while still over-privileged (e.g. a build+release pair where only the release job needs write). If you spot one, tighten it: root `contents: read`, elevate on the job that needs it. This is proactive — it may not move a moat count — but it's the same least-privilege principle, so do it while you're in the file.

Preview each workflow's changes (one preview per file, batching all the permissions blocks in that file — and combining with any pinning changes for the same file, per the batching rule in the guardrails), confirm, apply.

## Other possible findings

`moat` evolves. If the filtered findings include an `id` not handled above, **do not silently skip** — show the user moat's `label`, `description`, `why_enable`, and `how_to_fix` URL, and ask whether they want help with it. If the fix is a file change, work through it with the same preview-then-confirm pattern.

## Supply-chain hardening (proactive — not a moat finding)

moat does **not** flag anything in this section. It's proactive hardening, prompted by the run of supply-chain attacks across package ecosystems (the Sept 2025 `chalk`/`debug` compromise, the Shai-Hulud worm, the 2026 `axios` incident). Because moat didn't ask for it, it's **opt-in**: only offer it when the repo actually has the relevant files, and only proceed once the user says yes. Re-running moat afterwards will **not** show any of this as resolved — it was never a finding (see *Re-verify* in the closing summary).

**Dependabot is GitHub-only — mind the host.** Several guides below pair their one-off fix with an *ongoing* Dependabot twin (the npm/composer/gomod/docker cooldowns, automated base-image bump PRs). Those only run on **GitHub**. On a **GitLab-only** repo (no `github.com` remote) a `.github/dependabot.yml` does nothing, so that automated-freshness half is **inert** — the one-off pins you apply get no automatic upkeep, which is exactly the "pinned and forgotten → reproducibly vulnerable" trap `pinned-versions.md` opens with. Say so plainly rather than implying Dependabot has it covered, and point at the GitLab equivalents instead: **GitLab Dependency Scanning** (vulnerability detection) and **self-hosted Renovate** (the closest thing to Dependabot's version-bump MRs on GitLab). If neither is in play, the honest fallback is the dated pin-comments plus periodic manual review. (This is why the per-guide Dependabot notes carry a "GitHub-only" pointer back here.)

**These ecosystems are not five copies of the npm pattern.** Each maps the same axes through different mechanisms, and different things are already safe by default. The per-ecosystem detail lives in a self-contained guide under `assets/hardening/` — load only the one(s) that match the repo, and follow it. Orientation map (verified mid-2026 — the tools move fast, so re-check current docs before asserting a version or flag):

| Axis | npm | Composer | pip | uv | Go | Bun |
| --- | --- | --- | --- | --- | --- | --- |
| **Release-age cooldown** | `.npmrc` `min-release-age` (npm ≥ 11.10.0) | none native yet → Dependabot only | `--uploaded-prior-to` (pip ≥ 26.1) | `exclude-newer` (date or rolling age) | none native → Dependabot only | `minimumReleaseAge` in `bunfig.toml` (v1.3, seconds) |
| **Install-script lockdown** | `ignore-scripts` (opt-in) | dep scripts don't auto-run; `allow-plugins` deny-by-default (2.2+) | wheels-only `--only-binary :all:` | wheels-only `--only-binary :all:` | N/A — no install scripts (structural) | `trustedDependencies` allowlist (on by default) |
| **Locked install (the `npm ci` axis)** | `npm ci` | `composer install` (already is) | `--require-hashes` + pinned reqs | `uv sync --frozen` | `-mod=readonly` (default since 1.16) / vendor | `bun install --frozen-lockfile` |
| **Audit** | `npm audit` | `composer audit` + 2.10 malware policy | `pip-audit` | `uv audit` (preview) | `govulncheck` (reachability) | Security Scanner API (v1.3) |
| **Dependabot cooldown** | yes | yes | yes | yes (one known edge bug) | `default-days` only | yes (no security updates; no `bun.lockb`) |

**Detect, then load the matching guide(s).** A repo can trip several rows — a Laravel app has both `composer.json` and `package.json`. Offer each ecosystem in turn, one at a time. **And look beyond the repo root:** a marker file can sit in a subdirectory (a Go CLI at `cli/go.mod`, a frontend at `web/package.json`), so a root-only check misses it — scan subdirectories (skipping `vendor/` and `node_modules/`) and run the matching guide against the directory the marker actually lives in.

| Detected in the repo | Ecosystem | Guide to read |
| --- | --- | --- |
| `composer.json` | Composer | `assets/hardening/composer.md` |
| `pyproject.toml` / `requirements.txt` / `uv.lock` | Python (pip + uv) | `assets/hardening/python.md` |
| `go.mod` | Go | `assets/hardening/go.md` |
| `bun.lock` / `bun.lockb` / `bunfig.toml` | Bun | `assets/hardening/bun.md` |
| `package.json` + an npm/pnpm/yarn lock (no Bun signal) | npm (or pnpm/yarn) | `assets/hardening/npm.md` |
| `Dockerfile` / compose & stack files (`docker-compose.yml`, `*-stack.yml`) / `build.sh` / `Makefile` / workflow `image:`/`services:` (GitHub *and* `.gitlab-ci.yml`) | cross-cutting (image-digest pinning) | `assets/hardening/pinned-versions.md` |

**Package-manager guard.** `package.json` alone doesn't tell you the tool — disambiguate on the lockfile (`bun.lock`/`bun.lockb` → Bun; `pnpm-lock.yaml` → pnpm; `yarn.lock` → yarn; `package-lock.json` → npm). Load that ecosystem's guide; never write an npm `.npmrc` into a Bun/pnpm/yarn project.

All the usual guardrails apply: preview → confirm → apply, one change at a time, never a git mutation. Each guide carries its own sensible-default-plus-dial and previews before anything lands.

## Delivery (optional): open a GitLab merge request

Normally the user commits and pushes the applied fixes themselves (you never run git). When the repo's **primary remote** (from *Detecting the current repo*) is a GitLab/custom host — or the user asks to "raise an MR" / "open a gitlab MR" — you can *additionally* open a GitLab **merge request** via `glab`, on a `fix-moat-issues` branch so any tagged reviewer knows what it's about.

This step is **opt-in** and runs **after** the file fixes are applied and confirmed. In Mode B, ask **once** for the whole sweep (see Mode B), then apply per repo.

**The git boundary still holds.** Branch / commit / push are git *writes* — they stay the user's job (and are hook-blocked here). The only thing you run is `glab mr create`, which is a GitLab **API call** — the same category as the `gh api` mutations `moat-org-fixer` runs after a y/n confirm. Offer the same two modes:

- **Apply mode** — you run `glab mr create` after the user has pushed, on their y/n confirmation.
- **Hand-off mode** — you print the exact command and the user runs it.

**Never** pass `glab mr create --push`, and never run `git branch` / `commit` / `push` / `checkout` — pushing is the user's job (step 3).

> **Which default branch? The GitLab remote's own — never the GitHub default.** On a both-hosts repo the GitHub default branch (what moat scanned — and it can be an *accident* like `feature/save-form`, see *Reconcile the scanned branch*) is often **not** the GitLab default, and may not even exist on GitLab. Branching or targeting an MR against it would point at a non-existent branch. Resolve the GitLab default **independently**, once, and reuse it for both the sync pre-flight (step 2) and the MR target (step 4):
> ```
> git ls-remote --symref <gitlab-remote> HEAD   # prints: ref: refs/heads/<default>\tHEAD
> ```
> Read the branch name off the `ref: refs/heads/<name>` line — that `<name>` is the `<default-branch>` used below. Don't carry the GitHub default into delivery.

### 1. glab auth pre-flight

Mirror the `gh auth` gate in `moat-org-fixer`. `glab` must be authenticated against the repo's GitLab **host** (a self-managed instance, not `gitlab.com`):

```
glab auth status
```

The repo's GitLab host (from the remote URL) must appear as authenticated. If it doesn't, **halt the delivery step** (but keep the file fixes — auth only gates the MR) and tell the user:

> `glab` isn't authenticated against `<host>` (your self-managed GitLab). Run:
> ```
> glab auth login --hostname <host>
> ```
> then re-run the delivery step. (`glab` reads `GITLAB_HOST`/`GL_HOST` or the git remote to pick the host; for a self-managed server it must show in `glab auth status`.)

### 2. Sync pre-flight — is the default branch current with GitLab?

The developer may not have pulled recently. Check **read-only** — no `git fetch` (that's a write the hook blocks); `git ls-remote` only reads, so it passes:

```
git rev-parse <default-branch>
git ls-remote <gitlab-remote> refs/heads/<default-branch> | cut -f1
```

`<gitlab-remote>` is the remote name for the GitLab host (whatever it's called — `origin`, `gitlab`, … — keyed by host, not name); `<default-branch>` is usually `master` or `main`. If the two SHAs **match**, it's current — proceed. If they **differ**, stop and ask the user to pull first (you don't run git):

> Your local `<default-branch>` (`<localsha>`) differs from GitLab (`<remotesha>`). Pull first so the MR branches from current `<default-branch>`, then re-run the delivery step.

### 3. Branch, commit, push — the user's job

You can't run git writes, so hand the user the exact commands, using the `fix-moat-issues` convention:

```
git checkout -b fix-moat-issues
git add -A                                        # or just the files we changed
git commit -m "Apply moat security hardening fixes"
git push -u <gitlab-remote> fix-moat-issues
```

Wait for the user to confirm they've pushed — `glab mr create` needs the branch on the server.

### 4. Open the merge request

From **inside the repo** (so `glab` infers the project from the remote — no `-R` needed), once the branch is pushed:

```
glab mr create \
  --source-branch fix-moat-issues \
  --target-branch <default-branch> \
  --fill
```

- `--fill` takes the title/description from the commit(s); add `--title` / `--description` if the user wants something more specific (e.g. listing the moat findings actioned).
- `--target-branch` is the **GitLab remote's** default branch (resolved via `git ls-remote --symref` above — *never* the GitHub default, which may differ or not exist on GitLab); confirm `master` vs `main`.
- **Never** add `--push`; pushing was step 3.
- In **apply mode**, show the exact command and confirm (y/n) before running it; in **hand-off mode**, print it for the user.

`glab` prints the MR URL on success — surface it. On failure (auth, branch not pushed, host unreachable), show the error and stop; don't retry blindly.

**Verified against `glab 1.37.0`** — the flags above are long-standing, so they hold on newer versions too. A couple of self-managed host-resolution details weren't exercised live (the instance needed a VPN at design time): if `glab` can't resolve the host from the remote, set `GITLAB_HOST=<host>` for the command, or lean on `glab auth login --hostname`.

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

**3. Source the org's findings** (same preference order as Mode A, at org scope): if the user has a recent moat **report**, use it; otherwise offer to run `moat --format json <org>`. Keep only the **file-level** findings this skill owns — `repositories_workflow_actions_are_sha_pinned`, `repositories_workflow_permissions_are_restricted`, `repositories_have_security_policy`, `repositories_have_dependabot_config` (note the last two often show as **pass**: `have_security_policy` when an org default exists, `have_dependabot_config` when `github-actions` is already tracked). Settings-level findings — including `repositories_enforce_workflow_actions_sha_pinning` — belong to `moat-org-fixer`: name them in the summary but don't act on them here.

**4. Join findings to checkouts.** For each flagged repo, look it up in the map and bucket it:
- **one local checkout** → queue for fixing
- **no local checkout** → "clone needed" (list at the end; never clone automatically)
- **more than one checkout** (the dupe warning) → show the candidate paths and **ask the user which one** before queuing

**5. Fix each queued repo, one at a time.** For each, in turn:
- `cd` into its `local_path`.
- **Check the working tree first.** Show the repo's `current_branch` and `git status --porcelain`. If it's on a non-default branch or has uncommitted changes, **surface that and ask** whether to (a) fix here anyway, (b) skip this repo, or (c) stop the sweep. Never edit a dirty or feature-branch checkout silently.
- **Reconcile against the branch moat scanned** (the *Reconcile the scanned branch* step from the per-repo flow): compare the local `current_branch` to the repo's GitHub default branch and to the finding's `affected[].branch`. If they differ, the local checkout may not match what moat flagged — surface it and resolve before editing, exactly as in Mode A. In a sweep this is easy to miss across many repos, so check it per repo.
- Run the **per-repo flow above** for that repo, scoping the findings to that repo's entries (you already have them from step 3 — don't re-run moat per repo). The `SECURITY.md` org-default pre-check still applies — and in an org sweep there's very likely an org default, so expect to recommend skipping the per-repo file.
- The proactive **supply-chain hardening** section is part of the per-repo flow too, but it's opt-in and non-moat. In an org sweep, ask **once** whether to apply it to all qualifying repos (those with a `composer.json` / `package.json` / `go.mod` / `Dockerfile` etc.), then apply consistently per ecosystem — don't re-prompt for every repo.
- The optional **GitLab MR delivery** (see *Delivery*) is part of the per-repo flow too. If the repos' primary remote is a GitLab/custom host (or the user asked for MRs), ask **once** whether to open a `fix-moat-issues` MR per repo, then do it consistently — the auth and sync pre-flights run per repo, and `glab` infers each project from its checkout's remote (no `-R` needed). Branch/commit/push stay the user's job per repo; you only run `glab mr create`.
- Honour every guardrail: one fix at a time, preview → confirm → apply, never a git mutation.

**6. Closing summary (org sweep).** On top of the per-repo summaries, give an org-level roll-up:
- **Fixed** — repo → which files changed.
- **Skipped** — repo + reason (dirty tree, on a feature branch, user declined).
- **Clone needed** — flagged repos with no local checkout.
- **Ambiguous** — repos with multiple checkouts, and which path you used.
- **Settings-level (for `moat-org-fixer`)** — the org/settings findings you deliberately left alone.
- **Merge requests opened** — repo → MR URL, where GitLab delivery was used.
- **Commit/push is the user's job** — an org sweep leaves uncommitted changes across many working trees. Remind them clearly; you never run git. (With GitLab delivery, the user still branches/commits/pushes per repo; you only opened the MRs.)

## Guardrails

- Never run git mutations (`add`, `commit`, `push`, `checkout`, `reset`, `restore`, etc.).
- The GitLab *Delivery* step isn't an exception to that: `glab mr create` is a GitLab **API call**, not a git write, so it's allowed (after a y/n confirm, like `moat-org-fixer`'s `gh api` calls). The branch/commit/push around it stay the user's job, and **never** use `glab mr create --push`.
- Never bulk-apply across fix categories. One category at a time, one file at a time across categories.
- In **Mode B** (org sweep), also work one *repo* at a time — finish or skip a repo before moving to the next; never fan out edits across repos.
- **Within a single file**, batch multiple changes of the same kind into one preview and apply in one pass. Don't make the user confirm 11 nearly-identical line edits in one workflow.
- Always show a preview before writing — `diff -u` against a temp file for modifications, a fenced code block for new files.
- Already-correct items get skipped silently — don't rewrite a workflow file just to touch it.
- If you can't resolve something (e.g. a private action you don't have `gh api` access to), ask the user — don't guess a SHA.
- The **supply-chain hardening** section is proactive, not a moat finding — only offer it when the relevant ecosystem files (`composer.json`, `package.json`, `pyproject.toml`, `go.mod`, `bun.lock`, `Dockerfile`, …) exist, and only act on the user's say-so. Load the matching per-ecosystem guide; never write an npm `.npmrc` into a Bun/pnpm/yarn project.

## After all fixes

Print a structured summary with **all** of the following sections — don't skip any, even when empty. (In **Mode B**, this is the per-repo summary; Mode B's step 6 rolls these up across the org.)

1. **Files changed** — list each path that was written, with a one-line note on what changed.
2. **Skipped / deferred** — anything you didn't do and why (e.g. "action X is archived, recommended replacing before pinning").
3. **Out-of-scope findings (for `moat-org-fixer`)** — explicitly list every moat finding for this repo whose fix is a setting rather than a file. The user needs to see the *remaining* work, not just what was done. For each: the finding's `label` and the matching repo entry from `affected[]`.
4. **Next steps**:
   - The user needs to commit and push these changes themselves (you don't do git). **Tailor this to the primary remote** (from *Detecting the current repo*): a GitHub-primary repo gets the usual push to GitHub; a GitLab-primary repo gets the *Delivery* step instead — branch `fix-moat-issues`, push to GitLab, then the `glab mr create` MR (with its URL if you opened it) — not a "push to GitHub" instruction.
   - **If the repo has *both* hosts**, be explicit about *which change lands where* — don't leave the user assuming one push covered both:
     - **`.github/*` (workflows, `dependabot.yml`) only take effect on GitHub** — that's where Actions and Dependabot run; they do nothing on the GitLab side.
     - **`Dockerfile`, compose, and stack files are host-agnostic** — they take effect via whichever host builds/deploys, so they need to reach *both* hosts to matter on both.
     - **How they get there depends on the setup:** a *GitLab→GitHub mirror* may carry one push across automatically — say so if that's the case. But many both-hosts users **push to each remote by hand** (no mirror); for them, spell out the split above so the right files reach the right host. This manual-push case is separate from the GitLab-MR *Delivery* path — only walk *Delivery* if the user asked for an MR.
   - The org-level default-token fix in `moat-org-fixer` cascades to all repos and clears `repositories_actions_workflow_token_is_read_only` in one go — recommend it as a complementary safety net. But it does **not** clear `repositories_workflow_permissions_are_restricted` (the per-workflow `permissions:` block we fixed here): the two are different findings, so don't suggest deferring the file fix in favour of the org setting.
5. **Re-verify (optional)** — offer to re-run `moat --format json <owner>/<repo>` after the user has pushed. Be honest about the caveats: file-level checks (`workflow_actions_are_sha_pinned`, `workflow_permissions_are_restricted`, `have_security_policy`, `have_dependabot_config`) should flip — but only once the fixes land **on the branch moat scans** (the repo's GitHub default; see *Reconcile the scanned branch* — fixing a different local branch won't move them). Anything that reads GitHub-side state (branch protection, org policy) won't change until the matching settings are also updated. The proactive supply-chain hardening (per-ecosystem cooldowns, install-script lockdowns, locked installs, audit tooling, and the stale-pin checks) won't appear in moat output at all — moat doesn't check for any of these, so don't expect any count to drop; the protection is real, just invisible to moat.
