# claude-moat

Claude Code **skills** that *apply* the security fixes Laravel's [moat](https://github.com/laravel/moat) CLI flags. moat finds; these skills fix. `README.md` is the user-facing overview — this file is orientation for working *on* the repo.

## The one thing that will bite you: the dual-copy invariant

Each skill exists **twice**:

- `skills/<name>/` — the **tracked source**. This is what gets committed and published.
- `.claude/skills/<name>/` — the **live mirror** Claude Code actually loads in this repo.

They must stay **byte-identical**. When you change a skill, edit the source under `skills/`, then copy the changed files into `.claude/skills/`. Verify with:

```bash
diff -rq skills/<name> .claude/skills/<name>   # expect no output
```

`.claude/` and `.ant/` are both **gitignored**, so mirror edits never appear in `git status`. It's easy to update the source, see an almost-clean status, and forget the mirror is now stale. Always diff the two after editing a skill.

The reason for this is that you can edit the live skill that is available to Claude Code then when the tweaks are ready you can 'publish' them to the skills/ directory which is where users copy them from into their own Claude Code setups.

## Layout

```
skills/
  moat-repo-fixer/        file-level fixes, run inside a repo
    SKILL.md
    assets/
      dependabot/         base.yml | laravel.yml | php-package.yml  (cooldown-enabled)
      map_repos_to_local  bash resolver: GitHub repo name -> local checkout
      SECURITY.md         template
  moat-org-fixer/         settings-level fixes via `gh api` (Chrome DevTools MCP fallback)
    SKILL.md
```

A skill is a `SKILL.md` (YAML frontmatter + Markdown body) plus optional assets. The frontmatter `description` is the **trigger text** — it decides when the skill activates, so word it for the phrasings a user would actually type.

## House style when editing a SKILL.md

These conventions are deliberate and hard-won — keep them:

- **Preview → confirm → apply, one fix at a time.** Never bulk-apply. `diff -u` against an in-project `.moat-preview-*` copy for edits; a fenced code block for new files. Temp files live in the project dir, never `/tmp` (sandbox-proofing).
- **Never run git mutations** (`add` / `commit` / `push` / `checkout` / …). This is both the user's global rule and a load-bearing promise the skills make to their users — they review and commit.
- **Be honest about scope.** repo-fixer = file-level; org-fixer = settings. The npm supply-chain hardening in repo-fixer is **proactive and *not* a moat finding** — opt-in, detection-driven (`package.json` / `Dockerfile`), and it won't show up when moat is re-run. Don't let it masquerade as a finding-actioner.
- **Verify external-tool facts; don't assert from memory.** The skills' sharpest details came from checking docs/issues, not recall — e.g. the org-fixer's "2FA PATCH returns 200 but changes nothing" trap, npm's `min-release-age` needing ≥ 11.10.0, esbuild being degraded-not-broken under `ignore-scripts`. When adding guidance about `gh api`, npm, pnpm, or Dependabot, check it first.

## No build, test, or lint

The repo is Markdown plus one bash script. "Testing" a change means a **fresh-session review**: run the skill against a real or throwaway GitHub org and note where it gets stuck — most of the good edits came from exactly that. `assets/map_repos_to_local` is the only executable; sanity-check it standalone:

```bash
bash skills/moat-repo-fixer/assets/map_repos_to_local <org> <base-path>
```

## Where the *why* lives

Design decisions and pivots are recorded in the `ant` notebook (`.ant/ant.db`, gitignored — local only). Use the `ant` skill to read or add them, and `ait` for issues/tasks.
