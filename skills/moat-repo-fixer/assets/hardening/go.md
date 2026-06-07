# Go supply-chain hardening (proactive — not a moat finding)

Proactive, opt-in hardening for **Go** projects. moat does not flag any of this; re-running moat won't show it resolved — see `SKILL.md` › *Supply-chain hardening*. Guardrails: preview → confirm → apply, never a git mutation. Trigger on `go.mod`.

**This guide is deliberately light, and that's the honest answer — Go is structurally less exposed.** Go's own security team: *"It is an explicit security design goal of the Go toolchain that neither fetching nor building code will let that code execute, even if it is untrusted and malicious."* There is **no `postinstall` equivalent**, and the checksum database (`sum.golang.org` + `go.sum`) blocks post-publish tampering. So most of the npm baseline (install-script lockdown, `.npmrc`) **has no Go counterpart because the threat doesn't exist** — don't invent a knob.

**The one caveat to state, not omit:** *"There is no security boundary within a build: any package that contributes to a build can define an `init` function."* A dependency you actually import and compile still runs its `init()` / package code at runtime. Go removes the *fetch/build-time drive-by* (the worm-propagation vector), not the risk of malicious code you chose to compile in. *Unused* deps have no effect.

So the Go actions are mostly **confirm the on-by-default protections are still on**, plus two genuinely additive items.

## Confirm (don't blindly set): checksum & module protections

```
go env GOFLAGS GOSUMDB GONOSUMDB GOPRIVATE GOVCS GOTOOLCHAIN
```

- `GOSUMDB` defaults to `sum.golang.org` (on since Go 1.13). **Flag if `GOSUMDB=off`** — that disables checksum-DB verification.
- `GOPRIVATE` legitimately lists private module prefixes (which skip the sumdb). **Flag an over-broad value** (e.g. `*`) — it silently disables verification everywhere. Private modules *need* it, so flag, don't auto-narrow.
- `go.sum` must be committed and complete (an incomplete `go.sum` is a hard build error since Go 1.16).
- `go mod verify` checks the local cache **against `go.sum`** (not against the live sumdb — don't oversell it). Worth a CI step.

## Confirm: reproducible builds

- `-mod=readonly` is the **default since Go 1.16** — builds error rather than silently editing `go.mod` / `go.sum`. `GOFLAGS=-mod=readonly` in CI is belt-and-braces. This is Go's structural `npm ci` equivalent; there's no separate "ci" command.
- Vendoring (`go mod vendor`, used automatically when `vendor/` is present and the `go` line is ≥ 1.14) gives self-contained, in-tree-reviewable, offline builds. Cost: repo bloat and vendor churn. Optional.

## Fix (additive): `govulncheck` in CI

```
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

Reports known vulns, narrowed by **call-graph reachability** — only flags a vuln if the vulnerable symbol is actually reachable, so far fewer false positives than presence-only scanners. **Limits to state:** calls via `reflect` / `unsafe` can be missed; needs network to `vuln.go.dev`. Don't present it as exhaustive.

## Fix (additive): toolchain pinning

`GOTOOLCHAIN` (Go 1.21+) defaults to `auto` (downloads & verifies the toolchain named in `go.mod`). For reproducibility, pin an exact version in `go.mod`'s `toolchain` line, or set `GOTOOLCHAIN=local` to forbid auto-download. Cost: `local` errors instead of fetching a newer toolchain — the trade-off you're choosing.

## Release-age cooldown

No native Go equivalent. **Dependabot `gomod` supports `cooldown` with `default-days` only** — the per-semver `semver-*-days` keys are **not** honoured for gomod (official docs; several blogs get this wrong). Use:

```yaml
- package-ecosystem: gomod
  cooldown:
    default-days: 7      # semver-*-days are ignored for gomod
```

**GitHub-only:** Dependabot doesn't run on a GitLab-only repo — and since Go has *no* native cooldown, that leaves a GitLab-only Go project with no release-age cooldown at all. See `SKILL.md` › *Supply-chain hardening* for the GitLab story (Dependency Scanning / Renovate, or periodic review). `govulncheck` and the on-by-default checksum protections above are host-agnostic and still your best line of defence there.

---
*Sources / last verified ~mid-2026: go.dev/blog/supply-chain; go.dev/blog/go116-module-changes; go.dev/doc/go1.13; go.dev/doc/toolchain; pkg.go.dev govulncheck; GitHub Dependabot options reference. `GONOSUMCHECK` is not a current variable — don't recommend it.*
