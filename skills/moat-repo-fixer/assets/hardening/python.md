# Python (pip + uv) supply-chain hardening (proactive — not a moat finding)

Proactive, opt-in hardening for **Python** projects, covering both pip and uv. moat does not flag any of this; re-running moat won't show it resolved — see `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, one at a time, never a git mutation.

**Detect which tool is in use** before writing:

- `uv.lock`, or `pyproject.toml` with a `[tool.uv]` table → **uv** (use the uv settings below)
- `requirements.txt` / `requirements*.in`, or pip in CI / the Dockerfile → **pip**
- A project can use both — uv for dev, an exported `requirements.txt` for deploy. Handle each layer where it lives.

**The Python install-time threat:** installing a source distribution (sdist) runs its build backend (`setup.py` for legacy packages) — arbitrary code at install time. Wheels (`.whl`) are unpacked, not executed. There's no per-package "scripts off" switch like npm's; the lever is **prefer wheels**.

## Fix: wheels-only (the install-script analogue)

```
pip install --only-binary :all:        # pip
uv pip install --only-binary :all:     # uv
```

Refuses to build any sdist; installs only wheels. **What breaks:** any dependency with no wheel for your Python / OS / arch hard-fails ("no matching distribution") instead of silently compiling — common on Alpine/musl, ARM, brand-new Python releases, and some scientific packages. Targeted escape hatch: `--only-binary :all: --no-binary <trusted-pkg>` (accepting that the exception runs build code).

- **Don't** recommend `--no-build-isolation` as "hardening" — it *weakens* isolation. Build isolation is on by default and should stay on.

## Fix: hash-pinning / integrity

- **pip — `--require-hashes`:** forces every requirement (incl. transitive) to be pinned `==` *and* hashed; an unlisted/unhashed dep is a hard error. Generate with **`pip-compile --generate-hashes`** (pip-tools). This + a pinned, hashed `requirements.txt` is the de-facto `npm ci` equivalent for pip.
- **uv — on by default:** `uv.lock` records a SHA-256 per artifact and `uv sync` / `uv run` verify it, failing on mismatch. No flag needed for project workflows. For the pip interface: `uv pip compile --generate-hashes` + `uv pip install --require-hashes`.
- **What breaks:** hash mode is brittle by design — any unpinned transitive dep, or a differing platform wheel, breaks the install. Regenerate the lock/hashes whenever deps change.

## Fix: locked installs (the `npm ci` axis)

- **uv:** `uv sync --frozen` installs strictly from `uv.lock` without re-checking (the CI/Docker step); `uv sync --locked` (or `uv lock --check`) *verifies* the lock matches `pyproject.toml` and errors if stale. The two are mutually exclusive.
- **pip:** `pip install --require-hashes -r requirements.txt` against a `pip-compile`-generated file. (pip 26.1 added experimental PEP 751 `pip install -r pylock.toml` — flag it as experimental, don't rely on it.)
- **Dockerfile:** uv → `RUN uv sync --frozen --no-dev`; pip → `RUN pip install --require-hashes --no-deps -r requirements.txt` (only with a fully pinned file). Base-image / Python-version pins → `pinned-versions.md`.

## Fix: release-age cooldown

- **uv — `exclude-newer`:** ignores distributions uploaded after a cutoff. Accepts an RFC-3339 date **or a rolling duration** (`"7 days"`, `"P7D"`) — the duration form is a true age-based cooldown like npm's. Persist it:
  ```toml
  [tool.uv]
  exclude-newer = "7 days"
  ```
  Requires the index to expose PEP 700 upload-time metadata (PyPI does; private indexes may not). `exclude-newer-package = { … }` for per-package exceptions.
- **pip — `--uploaded-prior-to` (pip ≥ 26.1):** a date or `PnD` duration (e.g. `P7D`); only considers versions that have sat that long. *(Confirm the exact flag name against `pip --help` on the pinned version — taken from the 26.1 release notes.)*
- **Dependabot:** supports both `pip` and `uv` ecosystems. **Use `package-ecosystem: "uv"` for `uv.lock` projects.** Keep its `cooldown` in step with `exclude-newer` so Dependabot doesn't open PRs uv then refuses to lock. *(Known edge bug: the uv-ecosystem cooldown has fired prematurely for a 1-day-old dep — dependabot-core #14544.)*
  - **GitHub-only:** Dependabot doesn't run on a GitLab-only repo (see `SKILL.md` › *Supply-chain hardening*). Python is the least-affected ecosystem here, though — the *cooldown itself* is native (uv's `exclude-newer`, pip's `--uploaded-prior-to`) and works regardless of host; it's only the automated *bump-PR* twin you lose on GitLab. There, lean on the native cooldowns + `pip-audit`, or a GitLab equivalent (Dependency Scanning / Renovate) for the automated bumps.

## Fix: index hardening (dependency-confusion)

An attacker publishes a public package shadowing your private one.

- **pip:** `--extra-index-url` makes pip consider *all* indexes and possibly pick the public match — the classic confusion vector. Prefer a single trusted `--index-url` (a mirror that proxies PyPI) and avoid mixing public + private.
- **uv:** keep the default `index-strategy = first-index` (restricts each package to the first index that has it); pin internal packages to an explicit `[[tool.uv.index]]` with `explicit = true`.

## Fix: auditing

- **pip-audit** (mature): `pip-audit -r requirements.txt`, default service the PyPI advisory DB (`-s osv` for OSV). Official CI action `pypa/gh-action-pip-audit`. To audit a uv project, `uv export` to a requirements file first.
- **uv audit** exists but is **preview / unstable** as of uv 0.11.x — usable, but its interface may change; don't make it the sole gate. (Don't confuse it with `uv check`, which runs the `ty` type-checker.)

---
*Sources / last verified ~mid-2026: docs.astral.sh/uv (resolution, indexes, sync, Dependabot); pip secure-installs / build-system docs; pip 26.1 release notes (cooldown, pylock); pypa/pip-audit. Flagged unverified: `uv pip --verify-hashes`; the exact pip cooldown flag name; `uv audit` stability — check `--help` on the pinned version.*
