# go-review

Senior-developer code review for Go projects, as a Claude Code skill.

Catches the issues that `go vet`, `staticcheck`, and standard CI miss: the production-killer patterns that pass tests and ship to prod before exploding under load. Every finding has a severity, a confidence level, and — for BLOCKER/MAJOR — a "why this matters in production" rationale framed in real-world consequences (latency, leak, corruption, blast radius).

This is a code-quality review, not infrastructure conformance. For "does this service follow our scaffolding patterns?", you want a separate template or scaffolding tool; this skill focuses on whether the Go code is well written.

## Install

User-level (available across every Go project you touch):

```bash
mkdir -p ~/.claude/skills
git clone <this-repo> ~/.claude/skills/go-review
```

Project-level (only in this repo):

```bash
mkdir -p .claude/skills
git clone <this-repo> .claude/skills/go-review
```

Start a new Claude Code session and `/go-review` should appear in the skills list.

## Usage

| Invocation | What it reviews |
|---|---|
| `/go-review` | Current branch diff vs `origin/main` |
| `/go-review pr 1234` | Files changed in PR #1234 (uses `gh pr diff`) |
| `/go-review internal/auth/ cmd/server/main.go` | Just those paths |
| `/go-review all` | Every tracked `.go` file (warns if >50) |

Or in natural language: "review this Go code", "Go code review", "review my Go PR".

### Flags

- `--with-tests` — also run `go test -race ./...` on affected packages. Off by default (race tests are slow).
- `--include-low` — show LOW-confidence findings (hidden by default unless severity is MAJOR or higher).

### What it does, step by step

1. Detects Go version (`go.mod` + `go version`) and module layout (monorepo-aware — only scopes `go ...` to affected modules).
2. Runs `go vet`, `staticcheck` (if installed), `gofmt -l` / `goimports -l`. Cites their output; doesn't restate.
3. Reads every changed file in surrounding context — not just the diff hunk.
4. Applies the rubric in `references/*.md`, prioritized by change-risk (concurrency, auth, payments, persistence get deeper scrutiny than DTOs and migrations).
5. Emits a structured report.

By default the review is **incremental**: it surfaces pre-existing issues only if they're BLOCKER-severity, security-relevant, or directly impacted by the change. PRs don't get derailed by unrelated tech debt.

## Example output

```
internal/auth/session.go
  [BLOCKER][HIGH] :47  Goroutine leak — no ctx cancellation
    Why this matters: under sustained traffic, blocked goroutines accumulate;
    memory pressure and scheduler overhead climb until p99 latency spikes.
    Fix:
      ctx, cancel := context.WithCancel(parent)
      defer cancel()
      go worker(ctx, …)

  [MAJOR][HIGH] :91  http.Client missing Timeout
    Why this matters: one slow upstream stalls the calling handler
    indefinitely, exhausting goroutines and connection pools.
    Fix: client := &http.Client{Timeout: 10 * time.Second}

  [MINOR][MED]  :12  Return concrete *Store; accept interfaces, return concrete types
  [NIT][LOW]    :28  Prefer errors.Is(err, sql.ErrNoRows) over string compare

=== Package health ===
internal/auth:
  Concurrency:   Weak    (goroutine leak; missing ctx propagation)
  Errors:        Strong
  API:           Moderate
  Observability: Weak    (no trace span on auth path)

=== Design summary ===
The session module mixes transport and persistence concerns; the goroutine
spawned in NewSession outlives the constructor with no shutdown contract.

=== Things that look good ===
- Clean context propagation through the cache layer
- Strong error wrapping with %w throughout the storage package
- Bounded concurrency with errgroup.SetLimit in the batch worker
```

## What it reviews

Ten categories. The summary lives in `SKILL.md`; deep rubrics with bad/good code examples live in `references/`.

| Category | Reference | Production-killers |
|---|---|---|
| Concurrency | `references/concurrency.md` | `time.After` loop leak, context detachment from request handlers, I/O under lock, unbounded goroutine fan-out |
| Errors | `references/errors.md` | `defer Close()` masking write errors |
| API design | `references/api-design.md` | — |
| HTTP & resources | `references/http-and-resources.md` | `http.Get` / `DefaultClient` (no timeout), `DefaultServeMux` (pprof exposure) |
| Database | `references/database.md` | — |
| Performance | `references/performance.md` | Pointer-default vs value semantics (heap pressure), interface boxing in hot loops |
| Testing | `references/testing.md` | — |
| Security | `references/security.md` | — |
| Observability | `references/observability.md` | — |
| Architecture | `references/architecture.md` | `init()` doing non-trivial setup |

**Production-killer patterns** are the "silent killers" — issues that pass `go vet`, `staticcheck`, and CI but cause real production outages. They default to BLOCKER/MAJOR severity at HIGH confidence.

### Severity levels

- **BLOCKER** — correctness, data corruption, race/deadlock, leak under load, security vulnerability. Fix before merge.
- **MAJOR** — maintainability cliff, API misuse, scalability issue, observability gap that will bite in prod. Should fix.
- **MINOR** — idiom violation, readability concern.
- **NIT** — formatting or preference. Rarely worth listing.

### Confidence levels

- **HIGH** — clear evidence in the code under review.
- **MED** — strong idiomatic concern; alternative is clearly better.
- **LOW** — possible smell; depends on caller context or runtime conditions not visible in the diff. Hidden by default.

## Suppression

Matches the [golangci-lint convention](https://golangci-lint.run/usage/false-positives/) — existing nolint-aware tooling and reviewers already understand it:

```go
result := userToken == expectedToken //nolint:go-review // not user-controlled, by design
```

`//nolint:all` also suppresses go-review. Suppressed findings are dropped from the report, not demoted. Always include a reason — bare `//nolint:go-review` works but flags a separate MINOR.

## How it complements other tools

go-review **does not duplicate** the work of automated tools — it cites them and adds the senior-judgment layer on top.

- **`go vet` / `staticcheck` / `golangci-lint`** — go-review runs these (when available) and cites their findings rather than restating them. If a tool already flagged shadowing on line 42, go-review won't re-flag it.
- **Service-scaffolding skills or templates** — if your team maintains conventions for service structure (module paths, Helm charts, CI pipelines, Dockerfile shape), pair go-review with that. go-review reviews code quality; scaffolding tools verify infrastructure conformance — orthogonal concerns.
- **`/security-review`** — generic security review. go-review's security section covers Go-language-specific pitfalls only (parameterized SQL, `crypto/rand`, constant-time compare, path traversal). For broader application-security review, use `/security-review` alongside.

## Customization

Edit the rubric files in `references/` to add team-specific patterns, internal libraries, or domain-specific concerns. The convention each file follows is:

> **principle → why it matters in production → bad vs. good code → how to spot it in review**

Production-killer patterns go into a `## Production-killer patterns` section at the top of the relevant file; they default to BLOCKER/MAJOR at HIGH confidence.

`SKILL.md` is the entry point — keep it under 500 lines; push detail into `references/`.

## Roadmap

Possible v2 additions:

- **Benchmark-aware mode** — inspect `_test.go` benchmark files for allocs/op regressions vs base branch; flag suspicious benchmark omissions on perf-sensitive packages.
- **`golangci-lint` integration** — third tool source alongside `go vet` and `staticcheck`.
- **Per-team `.goreview.toml`** — severity overrides, custom skip patterns, opt-in LOW-confidence findings.

## Notes

- **Monorepo-aware** — multiple `go.mod` files? go-review scopes `go vet` / `go test` to affected modules only, never runs from monorepo root.
- **Generated code skipped** — `*.pb.go`, `*_gen.go`, `*_string.go`, anything under `mocks/` or `gen/`, and any file with `Code generated ... DO NOT EDIT`.
- **Go-version-gated rules** — loop-variable capture is flagged differently on Go <1.22 vs ≥1.22, since the language semantics changed.
- **No fabrication** — empty diffs, non-Go projects, and missing tools all produce honest "nothing to review" or "tool unavailable" messages rather than invented findings.


