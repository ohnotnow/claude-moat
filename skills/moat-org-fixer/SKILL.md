---
name: moat-org-fixer
description: Apply settings-level security hardening fixes that Laravel's `moat` CLI flags at the org or repo-settings level — org-wide workflow token defaults, secret scanning / push protection / dependabot defaults, branch protection or repository rulesets, per-repo workflow permissions, direct collaborator cleanup. Use when the user has a moat report (or wants to run one) covering an organization and wants to action the settings findings. Prefers `gh api` over Chrome automation for safety and auditability; falls back to chrome-devtools-mcp for UI-only items. File-level fixes inside individual repos belong to the sibling `moat-repo-fixer` skill.
---

# moat-org-fixer

Walks through the *settings-level* fixes moat (https://github.com/laravel/moat) flags — things that change a config in GitHub itself rather than a file in a repo. File-level fixes belong to the sibling `moat-repo-fixer` skill.

## Step 0: Source the findings

Same preference order as `moat-repo-fixer`:

1. **User-supplied JSON file** — load and use directly.
2. **Live `moat` run** — needs an org or user account. Run:
   ```
   moat --format json <account>
   ```
   **Always pass `--format json`** — without it moat writes ANSI-coloured terminal output that's painful to parse.

   **`moat` exits non-zero when any check fails.** Don't chain it with `&&` — `moat … && jq …` silently skips the parse step because moat "failed". Redirect to a file as its own statement, then parse: `moat --format json <account> > /tmp/moat.json; jq … /tmp/moat.json`.
3. **Fallback** — `moat` isn't installed: point the user at https://github.com/laravel/moat. Don't fabricate a checklist.

## Step 1: Auth pre-flight (before anything else)

`moat` needs more scopes than a normal `gh auth login` provides. The org-admin endpoints this skill uses require the same scopes even for *reads* (e.g. `gh api /orgs/<org>/actions/permissions/workflow` needs `admin:org`). **Check scopes before doing anything else.**

```
gh auth status
```

**Required scopes**: `admin:org`, `repo`, `workflow`.

If any are missing, **halt the whole flow** and tell the user:

> Your current `gh` token is missing required scopes (you have: `<current>`; need: `admin:org, repo, workflow`).
>
> To re-authenticate with the elevated scopes, run:
> ```
> gh auth login -s admin:org,repo,workflow -h github.com -w
> ```
> This will open a browser and trigger an OAuth + 2FA flow. After you complete it, re-invoke this skill.
>
> **Important:** these are powerful scopes. Once we're done with the moat work, revoke them — see the closing summary for the exact steps.

Do not try to run moat, list orgs, or anything else with elevated impact until scopes are confirmed.

## Step 2: Choose execution mode

Use `AskUserQuestion` up front:

> How would you like me to handle the actual `gh api` commands?
>
> - **Apply changes for you** — I run each command after your y/n confirmation, showing current state before the change. Recommended if you're happy delegating execution to me.
> - **Hand-off mode** — I generate the exact commands grouped per finding and explain them; you run them yourself in a terminal. Useful if you want to inspect or delegate to a colleague who has the scopes.

In **apply mode**: per-finding flow is `show current → propose change → confirm → execute → verify`.

In **hand-off mode**: per-finding flow is `show current → present commands in a fenced block with what they do → wait for the user to type "done", "skip", or "stop here"`. Read-only `gh api` GET calls (to show current state) are still fair game — they're informational, not mutating.

The user can switch modes mid-session by saying so.

## Step 3: Detect account kind

Check `account_kind` in the JSON:
- `organization` → org-level settings apply
- `user` → many org-level toggles don't exist; skip those findings rather than failing on a 404

Throughout this skill, substitute `<org>` with the actual org name from `.account` in the JSON.

## Step 4: Group findings by leverage

Order matters — do the highest-leverage fixes first because they invalidate downstream per-repo fixes:

1. **Org-wide settings** (one API call → all repos)
2. **Branch protection / rulesets** (one config per repo, often identical across repos)
3. **Per-repo settings** that may have been resolved by step 1 — re-run moat between groups
4. **Per-resource changes** (e.g. remove specific collaborator from specific repo)

After applying step 1 fixes, **suggest re-running `moat --format json <org>`** to refresh the picture before grinding through per-repo work — and let the re-scan tell you what *actually* cleared rather than assuming. Beware one common false hope: the org-wide workflow-token change clears `repositories_actions_workflow_token_is_read_only`, but it does **not** clear `repositories_workflow_permissions_are_restricted` — moat checks that one at the workflow-*file* level, so no org/repo *setting* will move it (see the trap in Step 4.5).

## Step 4.5: Known traps and friction warnings

Before walking each finding, internalise these — they came from real run failures and silent-success traps that *look* fine but aren't. Skipping this section will cost you either correctness or trust.

### High-friction findings — warn the user BEFORE asking y/n

A bare "Apply? y/n" is not enough for these. Surface the impact in your own words, *then* ask. Moat may not include a friction note in its output; that doesn't mean there isn't one.

| Finding | Impact you must surface |
| --- | --- |
| `organization_requires_two_factor` | **Members without 2FA are removed from the org on enable.** Run `gh api '/orgs/<org>/members?filter=2fa_disabled' --jq '.[].login'` first and show that list to the user. These are the people who will lose access. Highest blast radius in this skill. |
| `repositories_commits_are_signed` | Every developer must configure commit signing (GPG / SSH / S/MIME). Unsigned pushes to protected branches will be rejected. This is a substantial workflow change — don't apply without the team agreeing. |
| `repositories_release_branches_are_locked` | Direct pushes to release branches blocked; PR-only workflow enforced. Hotfix processes need rerouting via PR. |
| `repositories_release_branches_have_linear_history` | Merge commits forbidden — squash or rebase only. Can break tooling that produces merge commits (some release automation, some IDEs). |

If moat's `description` or `why_enable` text contains its own caveats, surface those verbatim too.

### Silent-success traps (you think you applied; you didn't)

| Endpoint / finding | Trap |
| --- | --- |
| `organization_requires_two_factor` via REST PATCH | Returns 200, value unchanged. **UI-only via REST.** Don't attempt PATCH — flag as UI follow-up immediately. |
| `/orgs/<org>/actions/permissions/workflow` with PATCH | Returns 404. Use **PUT with the full body** — endpoint isn't a partial-update. |
| `repositories_releases_are_immutable` | No REST endpoint at all. UI-only. |
| Org defaults flipping FAIL → WARN after enabling toggles | After flipping the simple `secret_scanning_enabled_for_new_repositories` / push protection / dependabot toggles, moat will often re-report as WARN: "default policy missing". This is moat checking for a **formal Advanced Security Configuration** (the `security_products` settings page) — a *separate* thing from the simple toggle, and UI-only. Surface explicitly: "The WARN is expected and points at separate UI work." Completing that UI config *does* clear these to pass (confirmed on a real run). Don't let a fresh agent think they failed. |
| `repositories_workflow_permissions_are_restricted` "shrinking" after the org token change | It usually **doesn't**. moat checks each workflow file's `permissions:` block, not the repo/org default-token setting, so the org-wide `actions/permissions/workflow` PUT (and its per-repo equivalent) leaves this count untouched — they move a *different* axis (`repositories_actions_workflow_token_is_read_only`). This finding is genuinely file-level → `moat-repo-fixer`. (Seen for real: an org token change left a 28-repo count completely unchanged.) |
| Org-wide default `SECURITY.md` (a `<org>/.github` repo) | Makes every repo *display* a policy in its Security tab — genuinely useful — but moat's `repositories_have_security_policy` reads each repo's **own** contents and still flags all of them. The org default will not move that finding. Tell the user the protection is real either way; if they want the green tick it's a per-repo file → `moat-repo-fixer`. |

### Plan-tier limitations

| Limitation | Workaround |
| --- | --- |
| Org-level repository rulesets (`/orgs/<org>/rulesets`) need **GitHub Team or higher**. Free plan returns 403. | Fall back to **per-repo branch protection** (`PUT /repos/<o>/<r>/branches/<b>/protection`). Covers the same four findings (signed, reviews, locked release, linear history), just per-repo instead of cascading. Probe first: `gh api /orgs/<org>/rulesets` — 403 means Free plan. |
| Advanced Security Configurations on Free plan | Cover public repos only. For private repos on Free, the simple per-new-repo toggles are the best available. List as "plan-tier limitation" in the summary, not a failure. |

## Step 5: Walk through each finding

For every finding: **show current state → show proposed change → surface friction (if any) → confirm (or hand off) → apply → report**.

**Don't re-fetch state you've already shown this session.** If you swept the org settings in one go at the start, link back to that — don't run another GET before every PATCH. Re-fetch only when the previous show is stale (e.g. after a prior fix in this run that may have changed it).

**Findings sharing an endpoint can be batched.** The three "default policy" findings (secret scanning, push protection, dependabot security updates) all hit `PATCH /orgs/<org>`. One confirmation, one PATCH covers all three — don't fragment into three separate confirms.

**Mixed-aspect findings**: some findings (e.g. `repositories_workflow_actions_are_pinned`) have both an org-level toggle component (here) *and* a file-level component (in `moat-repo-fixer`). Handle the org-toggle portion in this skill and **explicitly hand off** the file portion to the sibling skill — name it in the closing summary's sibling-work section so the user sees the remaining work.

### Org-wide defaults

#### `organization_requires_two_factor`

**UI-only via REST.** PATCH attempts return 200 with the value unchanged. Don't try.

**Highest blast radius in this skill.** Before recommending the user enable this:

1. List members currently without 2FA:
   ```
   gh api '/orgs/<org>/members?filter=2fa_disabled' --jq '.[].login'
   ```
2. **Show that list to the user explicitly** — these are the people who get removed from the org on enable. Recommend they confirm individually with each affected member before applying.
3. Provide the exact UI path in the closing summary:
   > org → Settings → Authentication security → "Require two-factor authentication for everyone..." → Save
   > URL: `https://github.com/organizations/<org>/settings/security`

#### `repositories_releases_are_immutable`

**UI-only — no REST endpoint exists.** Path:
> org → Settings → Repository defaults → Releases → "All repositories"
> URL: `https://github.com/organizations/<org>/settings/repository-defaults`

List under "Needs UI follow-up" in the summary, not as a failure.

#### `repositories_actions_workflow_token_is_read_only`

The biggest single-setting win — one call cascades to every repo that hasn't overridden.

```
# Current
gh api /orgs/<org>/actions/permissions/workflow

# Proposed
gh api -X PUT /orgs/<org>/actions/permissions/workflow \
  -F default_workflow_permissions=read \
  -F can_approve_pull_request_reviews=false
```

**This endpoint is PUT-only with the full body.** PATCH returns 404. The two `-F` flags above *are* the full body — there are no other fields on this endpoint. **The PUT returns an empty body on success** — that's not an error; confirm with a follow-up `GET /orgs/<org>/actions/permissions/workflow`.

This clears `repositories_actions_workflow_token_is_read_only` for every repo that inherits the org default. It does **not** clear `repositories_workflow_permissions_are_restricted` — that's a file-level check (the workflow's own `permissions:` block) and no token *setting* touches it. Don't promise the user it will.

#### `repositories_secret_scanning_is_enabled` / `repositories_secret_push_protection_is_enabled`

These pass per-repo but moat flags "default policy missing" — new repos won't inherit the setting.

```
# Current
gh api /orgs/<org> --jq '{secret_scanning_enabled_for_new_repositories, secret_scanning_push_protection_enabled_for_new_repositories}'

# Proposed
gh api -X PATCH /orgs/<org> \
  -F secret_scanning_enabled_for_new_repositories=true \
  -F secret_scanning_push_protection_enabled_for_new_repositories=true
```

#### `repositories_dependabot_security_updates_are_enabled`

Same pattern:

```
gh api -X PATCH /orgs/<org> \
  -F dependabot_security_updates_enabled_for_new_repositories=true
```

#### `repositories_private_vulnerability_reporting_is_enabled`

Per-repo, with a clean REST endpoint. moat usually reports this as "enabled by default org-wide, but N repos override it" — so the fix is per-repo on the named repos:

```
# Current
gh api /repos/<owner>/<repo>/private-vulnerability-reporting    # -> {"enabled": false|true}

# Enable
gh api -X PUT /repos/<owner>/<repo>/private-vulnerability-reporting

# Disable (per GitHub's REST docs — rarely needed)
gh api -X DELETE /repos/<owner>/<repo>/private-vulnerability-reporting
```

PUT/DELETE return an empty body on success — verify with the GET. Batch across the affected repos like the other per-repo fixes.

**Self-inflicted-failure gotcha:** a *newly created* repo does **not** reliably inherit the org PVR default. If you create a repo as part of these fixes — e.g. a `<org>/.github` repo to hold a default `SECURITY.md` — it can surface as a fresh `private_vulnerability_reporting` failure on the next scan (and bumps the repo count, e.g. 67→68). Enable PVR on any repo you create, then re-check.

### Branch protection / rulesets

Affects: `commits_are_signed`, `pull_requests_require_reviews`, `release_branches_are_locked`, `release_branches_have_linear_history`.

**Sanity-check moat's target branches before applying.** moat reports a `release_branches` list per repo, but in practice that's whatever it treats as the release branch — usually the default branch, which in some repos is a transient *feature* branch (real examples from one run: `feature/api-llm`, `upgrade/8.0`, `feature/save-form`). Locking or enforcing linear history on a short-lived feature branch is rarely intended. Show the user the actual branch names and confirm the targets — don't blindly feed moat's list into a ruleset.

**Prefer an org-level repository ruleset** over per-repo branch protection rules where the user doesn't already have legacy branch protection in place. Rulesets are newer, cascade across repos in one config, and are easier to maintain.

**Plan-tier probe first.** Org-level rulesets need GitHub Team or higher:
```
gh api /orgs/<org>/rulesets
```
A 403 means Free plan — fall back to per-repo branch protection (covers the same four findings, just no cascade). Don't waste a confirmation cycle on rulesets only to bounce mid-flow.

```
# Current rulesets (on Team+)
gh api /orgs/<org>/rulesets

# Specific ruleset
gh api /orgs/<org>/rulesets/<id>
```

If the user already has legacy per-repo branch protection, extend that rather than creating parallel rulesets that conflict. Don't force a migration; mention it as a future improvement.

**Critical**: `PUT /repos/<owner>/<repo>/branches/<branch>/protection` is **replace-semantics**, not patch. Always GET the current protection, merge in the requested changes, then PUT the merged result. Never PUT a fresh config that doesn't include the user's existing rules — that nukes them silently.

**Nested bodies need `--input -`, not `-F` flags.** Branch protection's `required_pull_request_reviews` is a nested object; `gh api -F` can't form-encode it. Use a heredoc:

```
gh api -X PUT /repos/<o>/<r>/branches/<b>/protection --input - <<'JSON'
{
  "required_status_checks": null,
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1
  },
  "restrictions": null,
  "required_linear_history": true,
  "required_signatures": true
}
JSON
```

For batched application across N repos that all need the same rule:

1. Show the rule definition (the JSON body).
2. Show the list of N repo names.
3. **One confirmation** for the batch ("apply this rule to all N?").
4. Apply in parallel — fire all `gh api` calls in one tool message.
5. Report failures grouped at the end (repo + error message).

### Per-repo default-token overrides

The org-wide PUT (above) covers every repo that *inherits* the org default. A repo that has explicitly **overridden** its default token to `write` still shows under `repositories_actions_workflow_token_is_read_only` after the org change — fix those stragglers per-repo:

```
gh api -X PUT /repos/<owner>/<repo>/actions/permissions/workflow \
  -F default_workflow_permissions=read \
  -F can_approve_pull_request_reviews=false
```

Batch the same way as branch protection.

**Don't confuse this with `repositories_workflow_permissions_are_restricted`.** That is a *different* check — moat reads each workflow file's `permissions:` block, not the default-token setting, so this endpoint will not clear it. It's file-level → `moat-repo-fixer`. (Verified the hard way: an org-wide token change left a 28-repo `workflow_permissions_are_restricted` count completely unchanged.)

### Direct collaborators

`repositories_have_no_direct_collaborators` — typically small numbers and qualitatively different: this removes a *person*'s access.

1. List current collaborators per affected repo:
   ```
   gh api /repos/<owner>/<repo>/collaborators
   ```
2. Identify which are "direct" (`role_name` of `admin`/`write`/`triage`/`maintain` and not via team) versus inherited from teams.
3. **Per-person confirmation** before removal. Show username, role, and last-active info if you can get it.
4. Remove: `gh api -X DELETE /repos/<owner>/<repo>/collaborators/<username>`
5. If the person legitimately needs access, mention they should be added to a team — but **don't add them yourself**. Team membership is a separate decision.

### Unknown / new findings

`moat` evolves. For any setting-class finding ID not handled above:

1. Show moat's `label`, `description`, `why_enable`, and `how_to_fix` URL.
2. Look up the relevant GitHub API endpoint. If a clean `gh api` command exists, propose it through the standard preview→confirm→apply flow.
3. If it's genuinely UI-only, fall to the Chrome path below.
4. **Never silently skip** an unknown finding — surface it and let the user decide.

## Chrome fallback

For genuinely UI-only fixes (or anything you can't find the right API call for):

1. Check whether `chrome-devtools-mcp` is available — look for tools matching `mcp__chrome` or `chrome-devtools` in your tool list.
2. **If not available**, tell the user:
   > This fix needs Chrome DevTools MCP, which isn't enabled in this session. Exit and resume with `claude --chrome -c` — you won't lose the conversation. Then re-invoke this skill.
3. **If available**, drive the browser:
   - Navigate to the moat `how_to_fix` URL with `{org}` substituted
   - Screenshot before
   - Describe each click and **confirm before clicking** (apply mode) or wait for the user to click themselves (hand-off mode — just describe the navigation)
   - Screenshot after
   - On any unexpected page (login required, permission denied, page changed), stop and surface to the user — don't guess

## Guardrails

- **Never** touch org membership additions, billing, SSO config, audit log settings, or anything moat doesn't flag. Even if a finding *looks* like it points there — surface to the user and stop.
- **Never** modify branch protection without first showing the current rule. PUT is replace-semantics; getting this wrong silently removes existing protections.
- **Destructive per-resource** actions (remove collaborator, replace existing branch protection): confirm per-resource, not per-batch.
- **Identical config changes across N repos**: one confirmation per `(rule, repo list)` pair is fine — but show the list, don't hide it behind a count.
- Never use Chrome where `gh api` would work. The API is auditable, reversible, and leaves a trail; the browser doesn't.
- Same git rule as the repo skill: **never** run `git add`/`commit`/`push`/anything that mutates git state.
- Treat the `admin:org` scope window as a security risk — minimise time spent with it active. Don't tangent into unrelated org admin work just because the scope is there.

## After all fixes

Structured summary with **all** sections (don't skip empty ones — say "(none)" instead). Each section is qualitatively different; never merge them.

1. **Org-wide settings changed** — what, with the exact API command used.
2. **Per-repo changes applied** — which repos, what changed, count.
3. **Failures** — any API errors grouped by repo + error message.
4. **Needs UI follow-up (you must do these manually)** — every UI-only item with the **exact path, URL, and click sequence**. *No vague references.* "Settings → security_products" by itself is not enough — give the full URL (`https://github.com/organizations/<org>/settings/security_products`) and the click path. If the user doesn't know what `security_products` is, that's a skill failure, not a user failure.
5. **Skipped by user** — findings the user declined, each with a one-line note on how to re-enable later (the gh api command or UI path). Don't make them re-discover the fix.
6. **Sibling-skill territory (file-level)** — file-level findings that belong to `moat-repo-fixer`, listed per repo with affected files if obvious. Explicit handoff, not a one-line aside.
7. **Plan-tier limitations** — anything blocked by GitHub plan (e.g. rulesets on Free, Advanced Security Configurations on Free for private repos). Tell the user this is not their config to fix.
8. **Re-verify** — strongly recommend re-running `moat --format json <org>`. Most settings changes show immediately. **Expect FAIL→WARN downgrades** on the "default policy" findings — that's the formal Advanced Security Configuration vs simple toggle distinction (Step 4.5). Tell the user this is expected, not a fresh problem.
9. **Revoke elevated scopes** — print this exactly:
   > **Important: revoke the elevated scopes now that you're done.**
   >
   > 1. `gh auth logout`
   > 2. `gh auth login` (re-login with your normal scopes)
   > 3. Revoke the GitHub CLI authorization at https://github.com/settings/applications — find "GitHub CLI" and click "Revoke" if you don't routinely need it.
   >
   > The `admin:org` scope is powerful; leaving it on a long-lived token is the security smell `moat` exists to catch.
