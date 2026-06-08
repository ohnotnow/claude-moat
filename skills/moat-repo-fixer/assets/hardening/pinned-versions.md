# Stale & unpinned image versions — Dockerfiles, compose/stack files, CI (proactive — not a moat finding)

Proactive, opt-in hardening, **cross-cutting** (any ecosystem). moat does not flag this; re-running moat won't show it resolved — see `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, one change at a time, never a git mutation. Offer when a `Dockerfile`, compose/Swarm **stack** file (`docker-compose.yml`, `*-stack.yml`), `build.sh`, `Makefile`, or a workflow with `image:`/`services:` tags (GitHub Actions *or* `.gitlab-ci.yml`) is present.

**Why this belongs here.** This skill preaches pinning (SHA-pin actions, lock installs, cooldowns) — but a pin set two years ago and forgotten is *reproducibly vulnerable*: `FROM node:14` rebuilds the same pile of unpatched CVEs every time. Pinning is right; the cost of pinning is periodic review. This is that review. The classic drift: a dev runs `composer update` / `npm update` in the project and forgets the Dockerfile they wrote two years ago pins the toolchain separately.

## The iron rule: resolve "current" live — never bake a version into this skill

If this guide named "current" versions, it would become the very stale artefact it's warning about. So: **detect the pin → resolve the latest stable from the authoritative source at run time → flag → offer to apply.** Never carry a version number in the skill or assert one from memory.

Resolve live, e.g. (verify the command/endpoint still works before relying on it):

- Composer → `gh api repos/composer/composer/releases/latest --jq .tag_name`
- Go → `curl -s 'https://go.dev/dl/?mode=json' | jq -r '.[0].version'`
- Node / Python / `golang` base images → the official release feed / Docker Hub tags (`gh api` / registry); cross-check against the project's own support policy (e.g. Node LTS).

**Fetch lean — pull only the digest into context, not the manifest body.** Resolving an image digest can return a multi-arch manifest *list* (a sizeable JSON blob); reading the whole thing into context is a needless token cost, multiplied across many images and repos. Request just the digest — a registry `HEAD` reading the `Docker-Content-Digest` header, or a `--format` / `--jq` that emits only the `sha256:…` — so the ~71-char digest lands, not the manifest. (Verify the exact incantation still works; don't assert one from memory.)

**Reuse across a sweep — the resolution cache.** When hardening many repos the same images recur, so resolve each once via the optional `~/.cache/moat-repo-fixer/pins.tsv` cache described in `SKILL.md` › *Resolution cache* (48h TTL, write-checked + graceful-degrade, human still confirms the actual digest in the preview). It caches image digests here and action SHAs there.

## What to scan

**Dockerfile (structured — safe to act on):**

- Base images: `FROM node:14`, `FROM python:3.9-slim`, `FROM golang:1.20`, `FROM composer:2.6` → resolve current, flag the gap.
- Build ARGs: `ARG NODE_VERSION=18`, `ARG COMPOSER_VERSION=2.6.3`, `ARG GO_VERSION=1.20` → same.
- **Skip internal multi-stage stage refs.** A `FROM <earlier-stage>` (`FROM dev AS prod`, `FROM builder`) just references a stage defined earlier in the *same* Dockerfile — there's no upstream image or tag to pin, so leave it alone. Only the *external* bases (a registry image like `node:14`) get pinned. A multi-stage file commonly has a handful of these internal refs mixed in with one or two real bases; pinning a stage name is a mistake.
- **Variable / build-arg base tags need pinning where the value is set, not inline.** `FROM uogsoe/soe-php-apache:${PHP_VERSION}` has no literal tag to rewrite — the tag arrives from an `ARG PHP_VERSION` (often *supplied by CI*, e.g. a workflow passing `PHP_VERSION=8.0` as a build-arg). You can't pin the `FROM` line itself; trace where `PHP_VERSION` is defined (`ARG` default in the Dockerfile, or the `--build-arg` in the workflow / compose file) and flag *that* as the thing to pin/digest. Surface the indirection to the user rather than trying to rewrite the `FROM`.

**Compose / Swarm stack files & CI `image:` tags (structured — safe to act on):** `image:`/`services:` entries live in more than `Dockerfile`s. Scan:

- `docker-compose.yml`, and Swarm **stack** files like `prod-stack.yml` / `qa-stack.yml` (compose syntax, non-standard names): `image: redis:5.0.5`, `services: - mysql:5.7`.
- GitHub workflow `services:`/`image:` (e.g. a `services: { mysql: { image: mysql:5.7 } }`, or images passed to `docker-run`-style actions) and GitLab CI `image:`/`services:` in `.gitlab-ci.yml` (e.g. `image: docker:28`).

Pin the *third-party* ones to a digest. **Skip the project's own images** — those built in-pipeline and tagged by commit SHA (e.g. `$QA_IMAGE_NAME`, `${IMAGE_NAME}`); there's no upstream tag to pin and they're already content-addressed by the build.

**`build.sh` / `Makefile` (free-form — heuristic, surface for human review only, never auto-edit).** Flag obvious patterns and say plainly you can't infer intent from a shell script:

- image refs (`node:`, `python:`, `golang:`, `composer:` …)
- `*_VERSION=` assignments
- `go install x@v1.2.3`, `pip install foo==1.2.3`, `npm i -g tool@1.2.3`
- `curl …/download/vX.Y.Z/…`-style version-pinned fetches

## Moving and untagged tags are their own smell — the image-world `@latest`

Separate from a *stale* pin (a concrete-but-old `node:14`), watch for tags that aren't pinned to a version at all — the image equivalent of the `@latest` action smell:

- **Moving tags:** `docker:stable`, `node:latest`, `python:edge`, `:lts` — these are republished in place, so the same line pulls a different image over time. Non-reproducible, and a poisoned/regressed upstream lands silently. (`docker:stable` is itself *deprecated* on top of being a moving tag.)
- **Untagged images:** `image: mailhog/mailhog` with no tag at all resolves to `:latest` implicitly — same problem, easier to miss because there's no tag to even look at.

Flag these as a smell and resolve to a **concrete version + digest** (same comment convention as below). It's the direct parallel of recommending a tagged release over `@latest` for actions: pin to a real version first, then to its digest.

## Majors get a health warning — not a one-click

- **Patch/minor** (e.g. `node:22.1` → `node:22.7`, `python:3.12.1` → `3.12.7`): low-risk; offer as a normal preview → confirm edit.
- **Major** (e.g. `node:14` → `node:22`, `php:8.1` → `php:8.4`): potentially breaking. **Show the gap and the risk, recommend a deliberate, tested bump — don't present it as a safe one-click.** If they're far behind, step up to the nearest supported line first.
- **EOL images mean "upgrade *or* replace" — don't assume an in-place bump.** A base that's past end-of-life (`mysql:5.7`, EOL Oct 2023; `node:14`; `python:3.8`) gets no more security patches, so *pinning it to a digest just freezes the vulnerability* — flag it, don't pin it. But the fix isn't always "bump to a newer tag of the same image": sometimes the right move is to **switch to a different image entirely** (a team migrating `mysql` → `mariadb`, say). When you flag an EOL image, ask *"upgrade in place, or replace/migrate to a different image?"* rather than steering straight at a same-image version bump — and treat either as deliberate, tested work.
- Prefer pinning base images to a digest (`node:22-bookworm@sha256:…`) for reproducibility *and* letting Dependabot bump it (below). Resolve the digest live; never hand-write one from memory.

## Pinning to a digest: leave a comment that says what, when, and how to refresh

A bare `@sha256:…` is opaque — a human can't tell what version it is or whether it's stale. So every digest pin carries three things (a convention lifted from a real `prod-stack.yml`, originally a Bret Fisher tip): the human-readable tag, the date pinned, and the refresh recipe.

```yaml
# redis:5.0.5 — pinned 2026-06-05
# refresh: docker pull redis:5.0.5 && docker images --digests | grep redis
image: redis@sha256:f0957bcaa75fd58a9a1847c1f07caf370579196259d69ac07f2e27b5b389b021
```

This is deliberately richer than the actions convention (`uses: owner/repo@<sha> # v4`): image pins are less likely to be Dependabot-maintained, and the **date** is what lets both a human and this very check see when a pin has gone stale. Resolve the digest live; never hand-write one from memory. Adapt the comment syntax to the file — `#` works in Dockerfiles, compose/stack YAML, and CI YAML alike.

*Dual motive worth naming:* people pin images to digests for **availability** too — a Redis used as a session cache, pinned so it doesn't restart and log everyone out. Same mechanism, second payoff in supply-chain integrity. You don't have to sell it as a new trick.

## A deliberate, documented pin is *not* a smell — don't nag it

If a pin already carries a comment like the above — a named version, a date, a reason — treat it as **intentional**. Surface its rationale; do **not** reflexively recommend a bump. The redis example is pinned *on purpose* (bumping it restarts the session store and logs everyone out), and the comment is your only signal of that intent. Flag genuinely *undocumented* or obviously-rotted pins (a bare `FROM node:14` from years ago); leave documented deliberate ones alone unless the user asks — and if you do raise one, lead with the comment's stated reason, not the version gap.

**If the user asks to *match* a deliberate pin elsewhere, resolve its tag live first — it may have silently diverged.** A common request is "pin redis in dev/qa to the same digest `prod-stack.yml` already uses." Before copying the digest across, resolve the tag live: the pin may be an *older rebuild* of the same tag — same version (`redis:8.2.2`), different digest, because the upstream tag was rebuilt (an OS-layer patch) since prod was pinned. When that's the case, "match prod" and "be current" are no longer the same thing, so surface the three-way choice rather than copying blind:

- **match-old** — copy prod's existing digest for cross-environment consistency (what was literally asked);
- **current** — pin dev/qa to the freshly-resolved digest (newer, but now *ahead* of prod);
- **refresh-all** — bring prod up to current too.

And name the side-effect of touching the deliberate pin: refreshing it may carry the very deployment cost that froze it in the first place (refreshing prod redis restarts the session store → logs everyone out — see the dual-motive note above). Leave the call to the user; just make the divergence and its consequence visible.

## Don't let it drift again: Dependabot `docker` ecosystem

The one-off catch-up here pairs with the **ongoing** twin: Dependabot's `docker` ecosystem raises base-image update PRs automatically. The Dependabot fix in `SKILL.md` already adds `docker` when a `Dockerfile` is present — point the user at it so the Dockerfile stays current without manual sweeps. **Caveat for stack files:** Dependabot's `docker` ecosystem watches `Dockerfile`s and standard compose files, but *not* non-standard Swarm stack filenames like `prod-stack.yml` / `qa-stack.yml` without explicit `directory`/filename config — so digest pins in those files have **no automated freshness story** unless you add one (or accept they're reviewed by hand; the dated comment above is what makes that review possible). **And it's GitHub-only:** on a GitLab-only repo Dependabot doesn't run at all, so *every* digest pin here falls back to the dated-comment + manual-review story (or a GitLab equivalent — Dependency Scanning / Renovate); see `SKILL.md` › *Supply-chain hardening*. *(Confirm Dependabot's current base-image / digest / compose-filename behaviour before describing specifics.)*

---
*Sources / last verified ~mid-2026: GitHub Dependabot docker ecosystem docs; go.dev/dl JSON feed; Composer GitHub releases. The whole point of this guide is that versions are resolved live — it deliberately contains none.*
