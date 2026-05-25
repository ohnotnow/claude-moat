# claude-moat

Claude Code skills for actioning security findings from Laravel's [moat](https://github.com/laravel/moat).

## Status

Work in progress. This is a personal project; treat it as a starting point for your own customisation, not a production tool.

The `moat-org-fixer` skill asks for elevated GitHub scopes (`admin:org`, `repo`, `workflow`) and changes settings on your account or organisation. Test it on a throwaway organisation before pointing it at anything you care about. Mistakes here range from annoying (a tightened branch protection rule that breaks someone's hotfix workflow) to actively bad (a 2FA enforcement that boots collaborators out of your org without warning).

Use at your own risk. MIT licence, no warranties.

## What moat is

[moat](https://github.com/laravel/moat) is a new CLI from the Laravel team. It scans your GitHub account or organisation and reports security issues, with the steps to fix each one. Given how often supply-chain attacks now start with a compromised GitHub account, token or workflow, having something that flags the obvious gaps is useful.

moat tells you *what* to fix. The skills in this repository are about *applying* those fixes once moat has flagged them.

## What's in this repository

Two Claude Code skills, living under `skills/`.

### moat-repo-fixer

Handles in-repository, file-level fixes from a moat report. Run it inside the repository you want to harden. It walks through each finding one at a time, previewing the change before applying it. Covered fixes:

- Pinning GitHub Actions in workflow YAML to full commit SHAs, keeping the original tag as an inline comment.
- Generating a `dependabot.yml` config tailored to the detected ecosystem (composer, npm, github-actions, etc.).
- Adding a `SECURITY.md` from a template, with `git config` values offered as attribution defaults.
- Adding per-workflow `permissions: contents: read` to restrict the workflow token.

Before pinning, the skill also flags archived or stale third-party actions (`pushed_at` older than two years) so you do not accidentally pin a SHA from an abandoned action.

### moat-org-fixer

Handles organisation-level and repository-settings fixes via `gh api`, falling back to Chrome DevTools MCP for items that only exist in the web UI. What it does:

- Auth pre-flight. Refuses to start until the required scopes (`admin:org`, `repo`, `workflow`) are present on your `gh` token.
- Mode choice. Either "apply changes" (the skill runs each command after your confirmation) or "hand-off" (the skill produces the commands and you run them yourself).
- Leverage-ordered walkthrough. Org-wide settings first, then per-repo, with a recommended `moat` re-run between groups so per-repo failures that cascade from the org defaults clear themselves before you grind through them by hand.
- Known-traps section, written up from real-world testing. The two-factor auth PATCH that returns 200 without changing anything, the workflow-permissions endpoint that needs PUT rather than PATCH, and the FAIL-to-WARN downgrade pattern after enabling simple toggles.
- Friction warnings on high-impact findings. Signed commits, 2FA enforcement and locked release branches all get an explicit "here is what this breaks for your developers" note before the confirm step.
- A structured closing summary. Separate sections for UI follow-up (with exact paths and click sequences), user-skipped findings (with re-enable instructions), sibling-skill territory and plan-tier limitations.

Every successful run ends with a reminder to revoke the elevated scopes via `gh auth logout` and the [GitHub CLI authorisation page](https://github.com/settings/applications). An `admin:org` scope sitting in your shell config is exactly the kind of credential moat itself exists to flag.

## How it was built

The skills went through several real sessions before landing here. Each major change came out of a fresh-session review: a clean Claude run-through of the skill against a real or test GitHub organisation, then an honest look at where it got stuck. Fresh-session reviews surfaced the silent-success traps that hand-written design kept missing - the 2FA PATCH that returns 200 without doing anything, the `security_products` UI-only configuration, the GitHub Team plan requirement for org-level rulesets. None of that was in the first draft.

The skills are not finished. Call it a good first cut: useful, defensive, with rough edges. Contributions and feedback welcome.

## Prerequisites

You will need:

- [Claude Code](https://claude.com/claude-code).
- [moat](https://github.com/laravel/moat) installed and on your `PATH`.
- The GitHub CLI [`gh`](https://cli.github.com/), authenticated with `gh auth login`.
- `jq`, for filtering moat's JSON output.

For `moat-org-fixer` you will need to re-authenticate with elevated scopes when prompted:

```bash
gh auth login -s admin:org,repo,workflow -h github.com -w
```

The skill reminds you to revoke these scopes when you are done.

## Installation

Each subdirectory under `skills/` is a complete, ready-to-use skill. To install one:

1. Browse to the skill you want via the GitHub web UI (`skills/moat-repo-fixer/` or `skills/moat-org-fixer/`).
2. Copy the entire directory into either:
   - `~/.claude/skills/` for a personal, global skill, or
   - `<your-project>/.claude/skills/` for a project-local skill.
3. Restart your Claude Code session, or open a new one, to pick up the skill.

If you would rather track upstream changes, clone this repository and symlink the directories:

```bash
git clone https://github.com/ohnotnow/claude-moat.git
ln -s "$PWD/claude-moat/skills/moat-repo-fixer" ~/.claude/skills/moat-repo-fixer
ln -s "$PWD/claude-moat/skills/moat-org-fixer" ~/.claude/skills/moat-org-fixer
```

## Usage

Inside a Claude Code session both skills activate by description; no slash command needed. For example:

- "Run moat against my org and walk me through the org-level fixes." triggers `moat-org-fixer`.
- "Apply the moat findings to this repo, the report is at `./moat_report.json`." triggers `moat-repo-fixer`.
- "Help me pin my GitHub Actions to commit SHAs and add a dependabot config." triggers `moat-repo-fixer`.

Both skills accept either a user-supplied JSON file (the output of `moat --format json <target>`), or a live `moat` run that the skill kicks off itself. The repo-fixer auto-detects the current repository's owner and name.

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
