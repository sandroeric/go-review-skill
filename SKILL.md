---
name: go-review
description: Senior-developer code review for Go projects — concurrency correctness, error handling, API design, HTTP/server hygiene, database access, performance (hot-path only), testing, security, observability, and architecture. Findings include severity, confidence, and "why this matters in production." Triggers on "review this Go code", "Go code review", "review my Go PR", "/go-review", or invocation on .go file changes. Defaults to reviewing the current branch diff vs origin/main; supports a PR number, explicit paths, or `all`. Detects Go version; honors `//nolint:go-review` directives; skips generated code; respects monorepo module boundaries. Focuses on code quality, not infrastructure or service scaffolding. Skip for non-Go projects.
version: 1.0.0
---

# Go Code Review (Senior-Level)

## Purpose

Review Go changes the way a staff-level engineer would: catch the issues that pass `go vet`, `staticcheck`, and CI but still cause production outages. Output is calibrated for **senior judgment over checklist completeness** — every finding has severity, confidence, and a production-relevant rationale; LOW-confidence noise is suppressed by default.

This skill covers **code quality**: concurrency correctness, error handling, API design, resource hygiene, performance (hot-path only), tests, security, observability, and architecture. It does **not** cover infrastructure conformance (module paths, Helm charts, CI pipelines, Dockerfile shape) — that's a separate concern, typically handled by a service-scaffolding skill or template. Pair this skill with one of those if your team maintains those conventions.

## Workflow

Follow these steps in order. Do not skip; each subsequent step depends on the output of the previous.

### §0. Detect environment

Run these probes before reviewing anything. They control which rules apply.

1. **Effective Go version** — parse `go.mod` for the `go` directive and any `toolchain` line. Also run `go version`. Use the higher of the two as the effective version. Several rules below are version-gated (loop-variable capture, `context.WithoutCancel`, generics-as-an-alternative).
2. **Module layout** — find every `go.mod` file (`find . -name go.mod -not -path '*/vendor/*'`). If more than one exists, you are in a monorepo. Determine which modules contain changed files; scope every subsequent `go ...` invocation to those modules. Never run `go test ./...` from monorepo root reflexively.
3. **Tooling availability** — `command -v staticcheck`, `command -v golangci-lint`, `command -v goimports`. Record what is present. Never fabricate output from a missing tool.

### §1. Scope the review

Resolve the file list before running tools.

| Invocation | Scope |
|---|---|
| `/go-review` (no arg) | `git diff origin/main...HEAD -- '*.go'` |
| `/go-review pr <N>` | `gh pr diff <N>` filtered to `.go` |
| `/go-review <path> [<path>...]` | Those files / those packages |
| `/go-review all` | Every tracked `.go` file. **Warn if >50 files** and offer to narrow. |

**Skip generated files** (note that they were skipped in the report; emit no findings on their contents):

- Path patterns: `*.pb.go`, `*_gen.go`, `*_string.go`, anything under `mocks/` or `gen/`.
- Any file whose first 5 non-blank lines match `Code generated .* DO NOT EDIT`.
- Common generators to recognize: protoc, sqlc, ent, mockgen, stringer.

If the resulting scope is empty or contains no Go files, **return early** with a clear message: `No Go changes to review against origin/main.` Do not fabricate findings.

### §2. Run cheap static analysis first

Run these on the affected modules only. Capture outputs verbatim. **Cite, do not restate.**

- `go vet ./...` — always.
- `staticcheck ./...` — if available on PATH.
- `gofmt -l .` and `goimports -l .` (if available) — formatting drift.
- `go test -race ./...` — **skipped by default**. Run only when invoked with `--with-tests`. The skill still reads existing `*_test.go` files for coverage assessment without executing them.

Save the tool outputs. When a tool already flags something (e.g., `go vet` catches shadowing), the rubric review must not re-flag it — cite the tool finding and move on.

### §3. Read each changed file in context

For every changed file, load the surrounding function and its primary callers. Reviewing a hunk in isolation produces wrong findings. If a function calls into another file, read enough of that file to understand the contract.

### §3.5. Incremental review mode (default)

Findings target **changed lines and their immediate surrounding context**. Do **not** surface pre-existing issues unless one of the following holds:

- The issue is BLOCKER severity (race, leak, security, correctness, crash).
- The issue is security-relevant at any severity.
- The change directly impacts the pre-existing issue (e.g., your change exercises a previously-dormant code path).

This mirrors how senior human reviewers operate. The PR is the unit of change; unrelated tech debt belongs in a follow-up, not in this review.

### §4. Prioritize by change-risk

Spend more review budget on high-blast-radius code, less on low-risk code.

**Deep scrutiny** for changes touching:
- Concurrency primitives (`go`, `chan`, `sync.`, `atomic.`, `context.`)
- Auth, sessions, tokens, secrets
- Payments / billing
- Persistence (databases, queues, caches)
- Distributed coordination (locks, leader election, leases)
- Retries / timeouts / circuit breakers
- `unsafe.Pointer`, `reflect`
- Public / exported API surfaces

**Lighter scrutiny** for:
- Generated code (already skipped)
- Mocks and fakes
- DTO / serialization-only structs
- Migration snapshots
- Test fixtures

### §5. Apply the rubric

Walk through the 10 review categories below. For each category, the deep patterns and concrete code examples live in `references/<topic>.md` — read the relevant reference file before flagging anything in that category.

**Honor suppression directives** (matching the golangci-lint convention):

- `//nolint:go-review` — suppress go-review findings on the same line, or on the line immediately below if the comment is on its own line.
- `//nolint:go-review // <reason>` — same, with explicit justification. Reasons help future reviewers; flag suppressions without reasons as a separate MINOR.
- `//nolint:all` — suppress all linters (including go-review) on this line.

Suppressed findings are **dropped** from the report, not demoted to NIT.

### §6. Emit the structured report

Format defined in [Output format](#output-format) below.

## Review categories

Each category gets a 5–10 line summary here; the deep version lives in `references/<topic>.md`. Read the relevant reference file before producing findings in that category.

### Concurrency

Every `go func()` must have a clear exit path: ctx cancellation, closed channel, or finite work. `ctx` is the first parameter and is never stored in a struct. Channel ownership: only the sender closes; multi-sender requires coordination. Mutex scope is minimal and never spans IO. `WaitGroup.Add` happens before `go`, never inside the goroutine. **Loop-variable capture is Go-version-gated**: on Go <1.22, flag classic range capture in goroutines; on Go ≥1.22, only flag captures that involve pointers or shared mutation. Cover modern patterns: `errgroup` with `SetLimit`, **bounded fan-out over untrusted input**, `ticker.Stop()` / `timer.Stop()`, `context.WithCancelCause` where the cause is needed. See `references/concurrency.md`.

### Errors

Wrap with `%w`; reserve `%v` for terminal logging. Use `errors.Is` / `errors.As` over string comparison. No silent swallow (`_ = doThing()` without a comment explaining why). No double-handling: log *or* return, not both. Sentinel errors only when callers actually branch on them. Add context up the stack — the caller should know *what failed*, not just *that something failed*. See `references/errors.md`.

### API design

Accept interfaces, return concrete types. Small interfaces (>3 methods is a smell). No `Get` prefix on getters (`user.Name()` not `user.GetName()`). Receiver consistency on a type — don't mix value and pointer receivers. Useful zero values where possible. Options pattern over 5+ positional args. Avoid `util`, `common`, `helpers` package names. Minimize exported surface. See `references/api-design.md`.

### HTTP & resources

`http.Client` must have an explicit `Timeout`. Server `Read/Write/Idle` timeouts owned at one layer. `defer Close()` goes **after** the error check on bodies, rows, files. `http.MaxBytesReader` on untrusted request bodies. Response body drained on errors before close. Retry safety on non-idempotent methods (don't auto-retry POST/PATCH without an idempotency key). Middleware ordering matters (recovery first, then logging, then metrics, then auth). `io.ReadAll` only on bounded readers. See `references/http-and-resources.md`.

### Database

Always check `rows.Err()` after iteration. `defer tx.Rollback()` immediately after `BeginTx` — it's idempotent after a successful `Commit`. Use ctx-aware variants: `QueryContext`, `ExecContext`, `QueryRowContext`. N+1 query shapes get flagged. Missing `LIMIT` on user-driven queries is a blast-radius bug. Prepared statements reused across calls where applicable. See `references/database.md`.

### Performance

**Only flag when it matters.** A performance finding requires one of: request-path handler code, batch processing, a loop over unbounded input, allocations visible in a hot path, or existing benchmark/profile evidence. Otherwise downgrade to NIT or drop. Areas: slice/map pre-sizing, `strings.Builder`, `[]byte`↔`string` conversion costs, receiver consistency, reflection in hot paths, `sync.Pool` pitfalls. See `references/performance.md`.

### Testing

Table-driven with `t.Run` subtests. `t.Parallel()` where safe (no shared mutable state, no env vars). `t.Helper()` on assertion helpers. **Error paths tested**, not just the happy path. `testdata/` directory for golden files. Race tests on concurrent code. Benchmark hygiene (`b.ResetTimer`, `b.ReportAllocs`, realistic inputs). Meaningful test names (`TestX_WhenY_ThenZ`). See `references/testing.md`.

### Security

Parameterized SQL — never `fmt.Sprintf` user input into a query. `crypto/rand` for tokens, secrets, IDs; `math/rand` is **never** acceptable for anything security-relevant. `subtle.ConstantTimeCompare` or `hmac.Equal` for secret comparison. `filepath.Clean` + an explicit boundary check on user-supplied paths. `MaxBytesReader` on untrusted bodies. Default `http.Client` has no timeout — refuse it. TLS config must meet minimums (TLS 1.2+, sane cipher suites). See `references/security.md`.

### Observability

Structured logging (`log/slog` or `zap`) over `fmt.Printf` / `log.Printf`. **Never** log secrets, tokens, full request bodies, or PII. Request and trace IDs propagated through context. Metrics around retries, failures, timeouts (counter + histogram, not just counter). Error causality preserved (`%w`) so logs show the chain. No duplicate logging at multiple layers — the layer closest to the cause logs it, the rest just propagate. See `references/observability.md`.

### Architecture

Hidden global state is a smell. Cyclic package dependencies are a refactor signal. God packages (one package doing 5 things) split. Transport / business logic separation maintained (HTTP handler is thin, business logic is testable in isolation). Repository pattern doesn't leak into handlers. `context.Context` propagates across every layer boundary. Retry logic at exactly one layer (not three). Timeout ownership at one layer — typically the boundary nearest the user. See `references/architecture.md`.

## Output format

Group findings by file, then by severity descending (BLOCKER → MAJOR → MINOR → NIT). Every line has severity, confidence, `file:line` reference, and a one-line summary. BLOCKER and MAJOR findings additionally include a **Why this matters** sentence framed in production terms (latency, leak, corruption, exhaustion, blast radius), and a **Fix** snippet when it meaningfully clarifies the resolution.

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
```

**Report footer** — always emitted, even on a clean report:

```
=== Package health ===
internal/auth:
  Concurrency:   Weak    (goroutine leak; missing ctx propagation)
  Errors:        Strong
  API:           Moderate
  Testing:       Moderate
  Observability: Weak    (no trace span on auth path)

=== Design summary ===
2–4 sentences on architecture / API concerns that span files.

=== Things that look good ===
- Clean context propagation through the cache layer
- Strong error wrapping with %w throughout the storage package
- Bounded concurrency with errgroup.SetLimit in the batch worker
```

Package health uses **Strong / Moderate / Weak** per category. Categories with no signal in the change are omitted (don't pad).

## Severity & confidence

### Severity

- **BLOCKER** — correctness, data corruption, race/deadlock, leak under load, security vulnerability, crash risk. Fix before merge.
- **MAJOR** — maintainability cliff, API misuse risk, scalability issue, observability gap that will bite in production, brittle ownership/lifecycle. Should fix.
- **MINOR** — idiom violation, readability, future-maintainability concern.
- **NIT** — formatting or preference only. Rarely worth listing.

### Confidence

- **HIGH** — definite correctness / security / concurrency issue with clear evidence in the code under review.
- **MED** — strong idiomatic concern; pattern is well-known and the alternative is clearly better.
- **LOW** — possible smell; depends on caller context, runtime conditions, or design intent not visible in the diff.

## Quality gates

Enforce these before delivering the report:

1. **Every BLOCKER and MAJOR has `file:line` + a "Why this matters" sentence** framed in production terms. If you cannot articulate the production consequence, the finding is not BLOCKER/MAJOR.
2. **Don't speculate.** A finding requires evidence: the code, the call graph, sync semantics, or tool output. If you cannot point to evidence, lower confidence to LOW, omit the finding, or convert it into a follow-up question to the user.
3. **LOW-confidence findings are hidden by default.** Include them only when severity ≥ MAJOR, or when the user opted in via `--include-low`.
4. **Don't restate tool output.** If `go vet` or `staticcheck` already flagged the issue, cite the tool ("`go vet`: shadowed variable on line 42") and move on.
5. **Honor `//nolint:go-review` and `//nolint:all`** — drop those findings, don't demote them.
6. **Empty diff or non-Go scope → return early.** Never fabricate findings to "have something to report."
7. **Cap at ~30 findings per file.** If exceeded, summarize and ask the user to narrow scope. A 200-finding report is a config problem, not a useful review.
8. **Production-killer patterns** (the patterns in the `## Production-killer patterns` subsection of each reference file) default to BLOCKER or MAJOR severity with HIGH confidence. Demote only with explicit evidence that the surrounding context neutralizes the concern.

## References

Deep rubric for each category lives in `references/`. Read the relevant file before producing findings:

- `references/concurrency.md` — concurrency patterns (4 production-killer patterns)
- `references/errors.md` — error handling (1 production-killer pattern)
- `references/api-design.md` — API design
- `references/http-and-resources.md` — HTTP and resource hygiene (2 production-killer patterns)
- `references/database.md` — database access
- `references/performance.md` — performance (2 production-killer patterns)
- `references/testing.md` — testing
- `references/security.md` — security
- `references/observability.md` — observability
- `references/architecture.md` — architecture (1 production-killer pattern)

Files marked with production-killer patterns start with a `## Production-killer patterns` section — these are the "silent killers" that pass standard CI and still cause outages. Treat them as BLOCKER/MAJOR HIGH-confidence by default.
