---
name: moat-repo-fixer
description: Apply in-repo security hardening fixes that Laravel's `moat` CLI flags for the current repository — pin GitHub Actions to commit SHAs, add a Dependabot config tailored to the repo's ecosystems, drop in a `SECURITY.md`, and restrict per-workflow token permissions. Use when the user is inside a repo and wants to action moat findings, or asks to "harden this repo", "fix moat issues here", "apply moat fixes". Settings-level fixes (branch protection, org defaults) are out of scope — those belong to the sibling `moat-org-fixer` skill.
---

# moat-repo-fixer

Walks through the in-repo fixes that moat (https://github.com/laravel/moat) flags. Strictly file-level changes inside the current repository. Settings-level fixes belong to `moat-org-fixer`.

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
4. Write to `.github/dependabot.yml`. Preview, confirm.
5. If `.github/dependabot.yml` already exists, diff the proposed against the existing and let the user decide what to merge.

### Fix: `repositories_have_security_policy`

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

Two routes:
- **Org-level** (single setting → fixes all repos at once) — belongs to `moat-org-fixer`. Mention this to the user and recommend it as the better long-term fix, but don't do it here.
- **Per-workflow** (in scope here) — add at the top of each `.github/workflows/*.yml`:
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

## Guardrails

- Never run git mutations (`add`, `commit`, `push`, `checkout`, `reset`, `restore`, etc.).
- Never bulk-apply across fix categories. One category at a time, one file at a time across categories.
- **Within a single file**, batch multiple changes of the same kind into one preview and apply in one pass. Don't make the user confirm 11 nearly-identical line edits in one workflow.
- Always show a preview before writing — `diff -u` against a temp file for modifications, a fenced code block for new files.
- Already-correct items get skipped silently — don't rewrite a workflow file just to touch it.
- If you can't resolve something (e.g. a private action you don't have `gh api` access to), ask the user — don't guess a SHA.

## After all fixes

Print a structured summary with **all** of the following sections — don't skip any, even when empty:

1. **Files changed** — list each path that was written, with a one-line note on what changed.
2. **Skipped / deferred** — anything you didn't do and why (e.g. "action X is archived, recommended replacing before pinning").
3. **Out-of-scope findings (for `moat-org-fixer`)** — explicitly list every moat finding for this repo whose fix is a setting rather than a file. The user needs to see the *remaining* work, not just what was done. For each: the finding's `label` and the matching repo entry from `affected[]`.
4. **Next steps**:
   - The user needs to commit and push these changes themselves (you don't do git).
   - For the workflow token permission specifically, the org-level fix in `moat-org-fixer` is one setting that cascades to all repos — they may prefer to defer the per-workflow fix and do that instead.
5. **Re-verify (optional)** — offer to re-run `moat --format json <owner>/<repo>` after the user has pushed. Be honest about the caveat: file-level checks (`workflow_actions_are_pinned`, `workflow_permissions_are_restricted`, `have_security_policy`, `have_dependabot_config`) should flip locally after push; anything that reads GitHub-side state (branch protection, org policy) won't change until the matching settings are also updated.
