# claude-moat

Claude Code skills for actioning security findings from Laravel's [moat](https://github.com/laravel/moat) - and for keeping the fixes fresh afterwards. Harden your repos, harden your org, then keep them hardened.

## Status

Work in progress. This is a personal project; treat it as a starting point for your own customisation, not a production tool.

The `moat-org-fixer` skill asks for elevated GitHub scopes (`admin:org`, `repo`, `workflow`) and changes settings on your account or organisation. Test it on a throwaway organisation before pointing it at anything you care about. Mistakes here range from annoying (a tightened branch protection rule that breaks someone's hotfix workflow) to actively bad (a 2FA enforcement that boots collaborators out of your org without warning).

Use at your own risk. MIT licence, no warranties.

## What moat is

[moat](https://github.com/laravel/moat) is a new CLI from the Laravel team. It scans your GitHub account or organisation and reports security issues, with the steps to fix each one. Given how often supply-chain attacks now start with a compromised GitHub account, token or workflow, having something that flags the obvious gaps is useful.

moat tells you *what* to fix. The skills in this repository are about *applying* those fixes once moat has flagged them - and then keeping them current, because a pin applied in January is quietly rotting by June.

## What's in this repository

Three Claude Code skills, living under `skills/`. The first two apply fixes - one at the file level, one at the settings level - and the third keeps those fixes from going stale, because hardening is not a thing you do once.

### moat-repo-fixer

Handles in-repository, file-level fixes from a moat report. Run it inside the repository you want to harden. It walks through each finding one at a time, previewing the change before applying it. Covered fixes:

- Pinning GitHub Actions in workflow YAML to full commit SHAs, keeping the original tag as an inline comment.
- Generating a `dependabot.yml` config tailored to the detected ecosystem (composer, npm, github-actions, etc.), including a `cooldown` so Dependabot will not raise PRs for day-zero releases.
- Adding a `SECURITY.md` from a template, with `git config` values offered as attribution defaults.
- Adding per-workflow `permissions: contents: read` to restrict the workflow token.

It also offers a round of **proactive supply-chain hardening** - prompted by the recent run of attacks across package ecosystems, and going beyond what moat itself flags. It is opt-in and detection-driven: the skill only offers what matches the files actually in the repository, with the per-ecosystem detail kept in reference files (`assets/hardening/`) it loads on demand. The ecosystems are deliberately not treated as copies of one another, because their threat models genuinely differ:

- **npm** - an `.npmrc` release-age cooldown (`min-release-age`) and install-script lockdown (`ignore-scripts`, checking for native-build packages first), plus switching `npm install` to `npm ci` in Dockerfiles.
- **Composer** - the real lever is `allow-plugins`, since a dependency's own scripts do not auto-run but its plugins do; `composer install` is already reproducible, and there is no native cooldown yet to lean on.
- **Python (pip + uv)** - wheels-only installs to avoid arbitrary `setup.py` execution, hash-pinning, frozen installs, and the release-age cooldowns both tools now have (`uv`'s `exclude-newer`, pip's `--uploaded-prior-to`).
- **Go** - deliberately light, and honest about why: Go runs no install scripts and its checksum database blocks tampering, so most of the npm playbook simply has no equivalent. Confirm the on-by-default protections, add `govulncheck`.
- **Bun** - already safe by default (its `trustedDependencies` allowlist is the inverse of `ignore-scripts`), with a native cooldown in `bunfig.toml`; the job is to review and tune, not add.

Alongside these, it checks any `Dockerfile`, `build.sh` or `Makefile` for **stale pinned versions** - an old `FROM node:14` or `ARG COMPOSER_VERSION=2.6.3` quietly rebuilding the same unpatched base every time. It resolves the current version live rather than baking one into the skill (which would only go stale itself), flags the gap, and treats a major-version jump as a deliberate decision rather than a one-click bump. The matching Dependabot `cooldown` - and its `docker` ecosystem, which keeps base images current - closes the same windows at the automated-PR layer.

Before pinning, the skill also flags archived or stale third-party actions (`pushed_at` older than two years) so you do not accidentally pin a SHA from an abandoned action.

It runs in one of two scopes, chosen at the start:

- **This repository** - the default. It auto-detects the current repo and walks its findings.
- **Every repository in an organisation** - a sweep across all your local checkouts of an org's repos, fixing each in turn, one repo at a time. This is where the bundled repo mapper earns its keep.

#### Mapping repositories to local checkouts

Local project folders rarely match their GitHub repository name - `devcheck` might live in a folder called `claude-bumblebee`, `repo-reporter` in `code_reporter_claude` - and the GitHub remote is not always called `origin` (a repo whose primary is an on-prem GitLab often keeps GitHub as a `github` backup remote). moat reports findings by GitHub repo name, but the repo-fixer needs the *local* path to work in, so the two have to be reconciled. Matching on folder names alone silently misses checkouts; matching on `origin` alone misses the GitLab-primary ones.

The skill ships a small bash resolver, `assets/map_repos_to_local`, to bridge that gap. It scans the immediate sub-directories of a base path, reads *every* git remote of each (never the folder name, and not just `origin`), and prints a CSV of the checkouts belonging to your organisation:

```bash
assets/map_repos_to_local UoGSoE ~/Documents/code
```

```
repo,local_path,current_branch,matched_remote,remote_url
devcheck,"/Users/you/code/claude-bumblebee",code-review,"origin","git@github.com:UoGSoE/devcheck.git"
examdb,"/Users/you/code/examdb",master,"github","git@github.com:UoGSoE/examdb.git"
```

The CSV goes to stdout (pipe it, or redirect with `> repo_map.csv`); a counts line and any duplicate-checkout warnings go to stderr. In the org-wide mode the skill runs this for you, then sorts each flagged repo into "fix it" (a local checkout was found), "clone needed" (no local copy) or "which one?" (the same repo checked out in two places), and checks each checkout's branch and working-tree state before touching anything. You can also run it standalone any time you just want the map.

### moat-org-fixer

Handles organisation-level and repository-settings fixes via `gh api`, falling back to Chrome DevTools MCP for items that only exist in the web UI. What it does:

- Auth pre-flight. Refuses to start until the required scopes (`admin:org`, `repo`, `workflow`) are present on your `gh` token.
- Mode choice. Either "apply changes" (the skill runs each command after your confirmation) or "hand-off" (the skill produces the commands and you run them yourself).
- Leverage-ordered walkthrough. Org-wide settings first, then per-repo, with a recommended `moat` re-run between groups so per-repo failures that cascade from the org defaults clear themselves before you grind through them by hand.
- Known-traps section, written up from real-world testing. The two-factor auth PATCH that returns 200 without changing anything, the workflow-permissions endpoint that needs PUT rather than PATCH, and the FAIL-to-WARN downgrade pattern after enabling simple toggles.
- Friction warnings on high-impact findings. Signed commits, 2FA enforcement and locked release branches all get an explicit "here is what this breaks for your developers" note before the confirm step.
- A structured closing summary. Separate sections for UI follow-up (with exact paths and click sequences), user-skipped findings (with re-enable instructions), sibling-skill territory and plan-tier limitations.

Every successful run ends with a reminder to revoke the elevated scopes via `gh auth logout` and the [GitHub CLI authorisation page](https://github.com/settings/applications). An `admin:org` scope sitting in your shell config is exactly the kind of credential moat itself exists to flag.

### bump-pins

The maintenance leg. `moat-repo-fixer` *creates* pins; this skill *refreshes* them. A pin is a snapshot, and a pin that is never bumped quietly becomes a liability - you end up reproducibly building the vulnerable version. `bump-pins` makes the refresh mechanical: resolve, preview, confirm, apply, then report what is still known-vulnerable. Run it in an already-hardened repo and it will:

- Bump pinned GitHub Action SHAs to newer releases via [pinact](https://github.com/suzuki-shunsuke/pinact), with a release-age cooldown so a freshly-poisoned release does not walk straight in. The cooldown has one deliberate exception: when the run is happening *because* a security patch just landed, that fresh release is the thing you came for, so the skill drops the cooldown for it and says so.
- Re-resolve Docker image digest pins - in Dockerfiles, compose and Swarm stack files, `.gitlab-ci.yml` and workflow `services:` - to whatever digest the tag currently points at. It never changes the tag itself: moving `node:22`'s digest forward is maintenance, moving `mysql:5.7` to `mysql:8` is a migration, and migrations stay human decisions. The same logic protects deliberately frozen pins - an EOL image kept as a conscious accepted risk gets listed once in the summary, not nagged about.
- Refresh the dated comment on each pin, since a stale "digest as of" date is worse than none - it actively lies about freshness.
- Finish with a report-only CVE pass over the exact pinned digests, via [trivy](https://trivy.dev/) (no account needed) or `docker scout` as a fallback. Container images get no Dependabot security alerts, only version PRs, so this pass is the genuine CVE channel for the image half. The report is interpreted rather than dumped: OS-layer noise is separated from findings in the app binary itself, and a build-only stage that ships nothing to production is weighed differently from a deployed service. Report-only means exactly that - a finding whose fix is a newer tag is surfaced as a decision for you, never applied.

Major-version action jumps get called out explicitly in the confirmation rather than waved through, for the same reason tags are sacred: majors change inputs and behaviour, and taking one should be a choice.

The conventions are the same as its siblings: preview-and-confirm one file at a time, graceful degradation when a tool is missing (it does the halves it can and says plainly what it skipped), and no git mutations - committing is yours. It runs against a single named pin ("bump the node pin"), the whole repo, or a fleet sweep across your local checkouts, reusing `moat-repo-fixer`'s repo mapper for the latter.

## How it was built

The skills went through several real sessions before landing here. Each major change came out of a fresh-session review: a clean Claude run-through of the skill against a real or test GitHub organisation, then an honest look at where it got stuck. Fresh-session reviews surfaced the silent-success traps that hand-written design kept missing - the 2FA PATCH that returns 200 without doing anything, the `security_products` UI-only configuration, the GitHub Team plan requirement for org-level rulesets. None of that was in the first draft.

The skills are not finished. Call it a good first cut: useful, defensive, with rough edges. Contributions and feedback welcome.

## Prerequisites

You will need:

- [Claude Code](https://claude.com/claude-code).
- [moat](https://github.com/laravel/moat) installed and on your `PATH`.
- The GitHub CLI [`gh`](https://cli.github.com/), authenticated with `gh auth login`.
- `jq`, for filtering moat's JSON output.

For `bump-pins` you will also want, depending on which halves you use:

- [pinact](https://github.com/suzuki-shunsuke/pinact) (`brew install pinact`) for the GitHub Actions half.
- Docker with buildx for the image-digest half.
- [trivy](https://trivy.dev/) (`brew install trivy`) for the CVE pass; `docker scout` works as a fallback if you are already logged into Docker Hub.

None of these are hard requirements - the skill does the halves whose tools are present and tells you what it skipped.

For `moat-org-fixer` you will need to re-authenticate with elevated scopes when prompted:

```bash
gh auth login -s admin:org,repo,workflow -h github.com -w
```

The skill reminds you to revoke these scopes when you are done.

## Installation

Each subdirectory under `skills/` is a complete, ready-to-use skill. To install one:

1. Browse to the skill you want via the GitHub web UI (`skills/moat-repo-fixer/`, `skills/moat-org-fixer/` or `skills/bump-pins/`).
2. Copy the entire directory into either:
   - `~/.claude/skills/` for a personal, global skill, or
   - `<your-project>/.claude/skills/` for a project-local skill.
3. Restart your Claude Code session, or open a new one, to pick up the skill.

If you would rather track upstream changes, clone this repository and symlink the directories:

```bash
git clone https://github.com/ohnotnow/claude-moat.git
ln -s "$PWD/claude-moat/skills/moat-repo-fixer" ~/.claude/skills/moat-repo-fixer
ln -s "$PWD/claude-moat/skills/moat-org-fixer" ~/.claude/skills/moat-org-fixer
ln -s "$PWD/claude-moat/skills/bump-pins" ~/.claude/skills/bump-pins
```

## Usage

Inside a Claude Code session the skills activate by description; no slash command needed. For example:

- "Run moat against my org and walk me through the org-level fixes." triggers `moat-org-fixer`.
- "Apply the moat findings to this repo, the report is at `./moat_report.json`." triggers `moat-repo-fixer`.
- "Help me pin my GitHub Actions to commit SHAs and add a dependabot config." triggers `moat-repo-fixer`.
- "Harden every repo in my UoGSoE org - they live under `~/Documents/code`." triggers `moat-repo-fixer`'s org-wide sweep.
- "The node security patch landed - bump the pin." triggers `bump-pins`.
- "Are our pins stale?" triggers `bump-pins`.

The two moat-driven skills accept either a user-supplied JSON file (the output of `moat --format json <target>`), or a live `moat` run that the skill kicks off itself. In single-repo mode the repo-fixer auto-detects the current repository's owner and name; in org-wide mode it asks for the organisation and the base path to your projects, then uses the bundled resolver to locate each checkout. `bump-pins` needs no moat report at all - it works from the pins already sitting in the repo.

## Testing it on your own setup

A throwaway organisation is worth the few minutes it takes to create. A reasonable fixture:

1. Create a new free GitHub organisation for testing.
2. Push two or three small repositories into it. A fresh `laravel new` works well.
3. Add a GitHub Actions workflow with deliberately unpinned actions (`uses: actions/checkout@v4` and so on) so the pinning fix has something to do.
4. Leave `SECURITY.md` and `.github/dependabot.yml` absent.
5. Run `moat --format json <test-org>` and feed the output to whichever skill you want to exercise.

Once you have walked the flow through on the throwaway org, dispose of it - or keep it around as a reusable fixture for re-testing when the skills change.

## Scope hygiene

If you authenticated with elevated scopes for `moat-org-fixer`, revoke them when you are done:

```bash
gh auth logout
gh auth login   # re-login with your normal scopes
```

Then visit https://github.com/settings/applications and revoke the GitHub CLI authorisation if you do not routinely need it.

A long-lived `admin:org` token is the kind of credential that ends up on a stolen laptop or in a leaked dotfile commit. The skill exists in part to catch exactly that pattern, so leaving one in your config afterwards is a bit silly.

## Contributing

Contributions welcome. The skills are plain Markdown (`SKILL.md`) with asset files alongside, so editing is straightforward.

To work on this repository:

```bash
git clone https://github.com/ohnotnow/claude-moat.git
cd claude-moat
# Edit the relevant skills/<name>/SKILL.md
```

There is no formal test suite. The most useful contribution is a fresh-session review: run a skill against a real GitHub setup, note the friction points, and open a pull request or issue with what you found. Several of the warnings in the skills came from exactly this process.

## Licence

MIT. See [`LICENSE`](LICENSE).
