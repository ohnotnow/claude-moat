# Stale pinned versions in Dockerfiles & build scripts (proactive — not a moat finding)

Proactive, opt-in hardening, **cross-cutting** (any ecosystem). moat does not flag this; re-running moat won't show it resolved — see `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, one change at a time, never a git mutation. Offer when a `Dockerfile`, `build.sh`, or `Makefile` is present.

**Why this belongs here.** This skill preaches pinning (SHA-pin actions, lock installs, cooldowns) — but a pin set two years ago and forgotten is *reproducibly vulnerable*: `FROM node:14` rebuilds the same pile of unpatched CVEs every time. Pinning is right; the cost of pinning is periodic review. This is that review. The classic drift: a dev runs `composer update` / `npm update` in the project and forgets the Dockerfile they wrote two years ago pins the toolchain separately.

## The iron rule: resolve "current" live — never bake a version into this skill

If this guide named "current" versions, it would become the very stale artefact it's warning about. So: **detect the pin → resolve the latest stable from the authoritative source at run time → flag → offer to apply.** Never carry a version number in the skill or assert one from memory.

Resolve live, e.g. (verify the command/endpoint still works before relying on it):

- Composer → `gh api repos/composer/composer/releases/latest --jq .tag_name`
- Go → `curl -s 'https://go.dev/dl/?mode=json' | jq -r '.[0].version'`
- Node / Python / `golang` base images → the official release feed / Docker Hub tags (`gh api` / registry); cross-check against the project's own support policy (e.g. Node LTS).

## What to scan

**Dockerfile (structured — safe to act on):**

- Base images: `FROM node:14`, `FROM python:3.9-slim`, `FROM golang:1.20`, `FROM composer:2.6` → resolve current, flag the gap.
- Build ARGs: `ARG NODE_VERSION=18`, `ARG COMPOSER_VERSION=2.6.3`, `ARG GO_VERSION=1.20` → same.

**`build.sh` / `Makefile` (free-form — heuristic, surface for human review only, never auto-edit).** Flag obvious patterns and say plainly you can't infer intent from a shell script:

- image refs (`node:`, `python:`, `golang:`, `composer:` …)
- `*_VERSION=` assignments
- `go install x@v1.2.3`, `pip install foo==1.2.3`, `npm i -g tool@1.2.3`
- `curl …/download/vX.Y.Z/…`-style version-pinned fetches

## Majors get a health warning — not a one-click

- **Patch/minor** (e.g. `node:22.1` → `node:22.7`, `python:3.12.1` → `3.12.7`): low-risk; offer as a normal preview → confirm edit.
- **Major** (e.g. `node:14` → `node:22`, `php:8.1` → `php:8.4`): potentially breaking. **Show the gap and the risk, recommend a deliberate, tested bump — don't present it as a safe one-click.** If they're far behind, step up to the nearest supported line first.
- Prefer pinning base images to a digest (`node:22-bookworm@sha256:…`) for reproducibility *and* letting Dependabot bump it (below). Resolve the digest live; never hand-write one from memory.

## Don't let it drift again: Dependabot `docker` ecosystem

The one-off catch-up here pairs with the **ongoing** twin: Dependabot's `docker` ecosystem raises base-image update PRs automatically. The Dependabot fix in `SKILL.md` already adds `docker` when a `Dockerfile` is present — point the user at it so the Dockerfile stays current without manual sweeps. *(Confirm Dependabot's current base-image / digest update behaviour before describing specifics.)*

---
*Sources / last verified ~mid-2026: GitHub Dependabot docker ecosystem docs; go.dev/dl JSON feed; Composer GitHub releases. The whole point of this guide is that versions are resolved live — it deliberately contains none.*
