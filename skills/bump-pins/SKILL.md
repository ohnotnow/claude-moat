---
name: bump-pins
description: Refresh the supply-chain pins that already exist in a repo — bump pinned GitHub Action SHAs to newer releases via pinact, re-resolve Docker image digest pins (Dockerfile, docker-compose and Swarm stack files, .gitlab-ci.yml, workflow services) to the tag's current digest, refresh the dated pin comments, and finish with a report-only docker scout CVE pass over the pinned images. Use whenever the user wants to update, bump, or refresh existing pins — "bump the node pin", "the security patch landed, update the pin", "refresh the digests", "update the pinned actions", "are our pins stale?", mentions pinact, or a security advisory lands for a pinned image or action. Creating *new* pins in an unhardened repo belongs to the sibling moat-repo-fixer skill; this skill keeps already-pinned repos current.
---

# bump-pins

Sibling to `moat-repo-fixer`: that skill *creates* pins; this one *refreshes* them. A pin that's never bumped quietly becomes a liability — you're reproducibly building the vulnerable version. This skill makes the bump mechanical: resolve, preview, confirm, apply, and tell the user what's now known-vulnerable.

Two halves plus a report: **actions** (pinact), **image digests** (docker imagetools), then a **CVE pass** (docker scout). Each half degrades gracefully if its tool is missing — do the halves you can, say plainly what you skipped and why.

## Prerequisites — check before starting

| Tool | Needed for | Check | Install |
| --- | --- | --- | --- |
| `pinact` | Actions half | `command -v pinact` | `brew install pinact` (Homebrew core) |
| `docker` (with buildx) | Digest half | `docker buildx version` | Docker Desktop |
| `trivy` | CVE pass (optional, no account needed) | `command -v trivy` | `brew install trivy` |
| `docker scout` | CVE pass fallback (needs Docker Hub login) | `docker scout version` | bundled with Docker Desktop |

If `pinact` is missing, tell the user the one-liner and offer to continue with just the digest half — don't reimplement pinact by hand with `gh api`; it handles tag→SHA resolution, version comments, and cooldowns better than ad-hoc calls will.

**GitHub API rate limits (pinact):** anonymous calls are rate-limited and a fleet sweep will hit the cap. Prefer pinact's keyring (`pinact token set`, then `PINACT_KEYRING_ENABLED=true` in the user's shell profile — *their* job to set, not yours). Do **not** `export GITHUB_TOKEN` or dump env vars — the user's hooks block it, and a token in the transcript is a rotation incident (it has happened; it cost days).

## Before touching anything

1. **Sweep stale previews** from any interrupted prior run: `find . -name '.pin-preview-*' -type f -delete`
2. **Check memories** (project memory and, if available, the user-memories MCP — search "moat") for deliberately frozen versions. Repos often pin EOL images (an old mysql, a redis that backs sessions) as a *conscious accepted risk*. This skill never changes tags anyway (see below), but don't nag about them either — list them once under "deliberately frozen" in the summary and move on.
3. **Ask scope** (AskUserQuestion): everything in this repo / just the named target ("bump the node pin" means node, not a full sweep) / a fleet sweep across local checkouts (reuse `moat-repo-fixer`'s `assets/map_repos_to_local` script and its Mode B orchestration — one repo at a time, ask policy questions once for the whole sweep).

## Preview discipline (same as moat-repo-fixer)

Never edit a real file directly. Copy it to `.pin-preview-<name>` next to the target, make the changes there (or point the tool at the preview copy), show `diff -u <original> <preview>`, get a yes, then `cp` back and remove the preview. On a no, just remove the preview. Batch all changes to one file into one preview — nobody wants to confirm twelve near-identical digest lines separately. **Never run git mutations** (`add`, `commit`, `push`, `checkout`, …) — committing and pushing is the user's job, always.

## Half 1: GitHub Actions (pinact)

For each `.github/workflows/*.yml` (and `action.yml` files if present):

1. Copy the workflow to its `.pin-preview-` file.
2. Run pinact against the preview copy — pinact accepts explicit file paths, which is what makes the preview dance possible:
   ```
   pinact run -update -min-age 7 <preview-file>
   ```
3. `diff -u` original vs preview. Read the diff before showing it:
   - **Major-version jumps** (the trailing comment shows `# v3.x` → `# v6.x`): call these out explicitly in the confirmation. Majors change inputs and behaviour (`actions/checkout` majors have dropped defaults before); the user may want to take the minor/patch now and the major deliberately.
   - **pinact exit code 2** means something it won't auto-fix (a branch ref, a missing version comment): surface it, don't force it. It *also* fires when a currently-pinned SHA is itself younger than `-min-age` (a "min-age violation" warning) — that's informational, not a failure; a freshly-hardened repo will trip it for a week. (Observed live, first run 2026-06-11.)
4. Confirm, apply, clean up — per file.

**The `-min-age 7` default and its exception:** the cooldown exists to dodge freshly-poisoned releases — most compromises are yanked within hours-to-days. But when the user is running this skill *because* a security patch just landed (the usual trigger!), a 7-day cooldown would refuse the very thing they came for. In that case drop `-min-age` (or scope with `-i '^owner/repo$'` to just the patched action) and say why: a release the user is deliberately, knowingly grabbing is the exception the cooldown rule anticipates.

## Half 2: Docker image digests

1. **Enumerate existing pins** — search tracked config for `@sha256:` (Dockerfile*, `*.yml`, `*.yaml`; skip `vendor/`, `node_modules/`, `.git/`). You're looking for refs of the form `name:tag@sha256:<digest>` in `FROM` lines and `image:` keys.
2. **Never change the tag — only re-resolve the digest the tag points at.** Moving `node:22`'s digest forward is maintenance; moving `mysql:5.7` to `mysql:8` is a migration. Tag changes are human decisions (and are sometimes deliberate freezes — see the memory check). Two shapes you can't re-resolve, so surface them instead:
   - a **bare digest with no tag** (`redis@sha256:…`) — there's no tag to follow; the human must choose one first;
   - a **build-arg tag** (`base-image:${VERSION}`) — unresolvable inline; note it and move on.
3. **Resolve each unique `name:tag`** (deduplicate first — the same image often appears in several files):
   ```
   docker buildx imagetools inspect <name:tag>
   ```
   The top-level `Digest:` line is the multi-arch index digest — the correct thing to pin (per-platform digests pin you to one architecture).
4. **Freshness check — the digest-world cooldown.** Before adopting a changed digest, check how fresh the new image is:
   ```
   docker buildx imagetools inspect <name:tag> --format '{{json .Image}}'
   ```
   Each platform entry carries a `created` timestamp. If the newest is under ~7 days old, warn before pinning — same poisoned-release logic as `-min-age`, same exception: if this fresh image *is* the security patch the user came for, that's a knowing grab, proceed. (Verified live 2026-06-11: this surfaced a node:22 rebuilt that same morning.)
5. **Edit the preview**: new digest, and refresh any dated comment on the pin (`# … digest as of YYYY-MM-DD`) to today — a stale date comment is worse than none, it actively lies about freshness.
6. One preview/diff/confirm per file, batching all that file's digest lines.
7. **Write-through to the shared cache** at `~/.cache/moat-repo-fixer/pins.tsv` (`image-digest <tab> name:tag <tab> sha256:… <tab> ISO-timestamp`), so the sibling skill's sweeps reuse what you just resolved. Run the append as its own command with an absolute path — chaining it after other commands has silently swallowed the write before (first run, 2026-06-11). If the cache isn't writable, skip silently — it's an optimisation, never a dependency. Re-verified-current digests are worth a fresh timestamped row too, not just changed ones — it renews their TTL for the sibling skill.

## CVE pass: trivy (report-only)

After the bumps (or even if nothing moved), give the user the "did I miss a CVE?" view that Dependabot can't — container images get no Dependabot security alerts, only version PRs, so this pass is the genuine CVE channel for the image half.

**Use trivy first** (`brew install trivy`) — it needs no account: the vulnerability DB downloads anonymously with a built-in `mirror.gcr.io` fallback (trivy ≥ 0.57.1). `docker scout cves` is the fallback for users already logged into Docker Hub — unauthenticated it refuses outright (verified 2026-06-11); never initiate a login yourself.

```
trivy image --severity CRITICAL,HIGH <name:tag>@<pinned-digest>
```

Scan the *exact pinned digest*, per unique image. For big images the table runs to hundreds of rows — add `-q` and extract the per-target `Total:` lines plus CRITICALs rather than dumping the lot.

**Interpret before reporting — raw counts mislead** (all verified live 2026-06-11):

- trivy reports per *target*: the OS package layer and each app binary separately. A full-fat Debian base (`node:22`) showed 287 CRIT/HIGH in the OS layer but only **1** in Node.js itself — the OS noise is real but mostly `affected`/`will_not_fix`/`fix_deferred` (the distro won't fix it in this release). Lead with the app-binary findings; summarize the OS layer.
- `Status: fixed` means a newer build clears it — that's a tag/version bump for the *user* to decide, not you. **Report only — never "fix" a finding by changing a tag.**
- Weigh where the image *runs*: a build-only stage (frontend asset builder) ships nothing of itself to production — its CVEs threaten the build environment, not prod. A deployed service (the QA mail catcher behind the proxy) is a different conversation. Say which is which.
- A slimmer base (`-slim`, `-alpine`) is the structural fix for OS-layer noise — worth one mention as an upgrade candidate, never a nag.

One scheduling note: anonymous Docker Hub pulls are per-IP rate-limited; a handful of images per run is nothing, but don't put an every-commit trivy-of-everything in CI without expecting throttling.

## Closing summary

Always include, even when empty:

1. **Bumped** — file → ref, old → new (short digests/SHAs), with the version movement where known (e.g. node 22.21.x → 22.22.3).
2. **Unchanged** — pins checked and already current.
3. **Surfaced for a human** — major-version action jumps offered but deferred, bare-digest pins with no tag, build-arg refs, anything pinact exited 2 on.
4. **Deliberately frozen** — known accepted-risk pins, listed without nagging.
5. **CVE report** — scout's critical/high per image, or "skipped: <why>".
6. **Push reminder** — the user commits and pushes; on dual-homed repos spell out the split: `.github/*` only matters once it reaches GitHub; Dockerfile/compose/stack/CI files take effect via whichever host builds and deploys (often GitLab), so they must reach both.
