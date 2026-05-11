# Architecture

Architecture findings are the highest-leverage outputs of a senior review. They span files, name design smells, and prevent the kinds of problems that linters cannot see because they live in the *relationships* between code, not inside any single file. Be deliberate: an architecture finding is a claim that something will be expensive to fix later. Don't reach for them on small PRs.

## Production-killer patterns

### `init()` doing non-trivial setup

**Why this matters in production.** `func init()` runs once per package at process start. It cannot return an error. Its execution order between packages is unspecified (only "before main"). It runs unconditionally — including under `go test`, even for tests that don't import the affected functionality, polluting the global environment of every test run.

When `init()` opens a DB connection, loads config, or registers HTTP handlers, the consequences are:

- **Initialization failures can only `panic`.** No graceful degradation, no retry, no fallback. A bad config file or an unreachable DB takes the whole process down at startup, often with a stack trace that doesn't mention the root cause.
- **Hidden dependencies.** Code that uses the side effects has no visible reason to import the package; you can't tell from a function signature what it actually needs.
- **Untestable.** Tests inherit whatever `init()` did. You cannot unit-test a service with a different config because the global was set at process start.
- **Order-dependent bugs.** When `init()` A depends on globals set by `init()` B, you have a latent bug — Go doesn't guarantee B runs first across packages.

**Bad:**
```go
package storage

var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        panic(err) // ← only way to handle errors in init
    }
}
```

**Good:**
```go
package storage

type Store struct {
    db *sql.DB
}

func New(ctx context.Context, dsn string) (*Store, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening postgres: %w", err)
    }
    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("pinging postgres: %w", err)
    }
    return &Store{db: db}, nil
}
```

The `main` function calls `storage.New(ctx, cfg.DSN)` and handles the error explicitly.

**Acceptable uses of `init()`:**
- Registering a SQL driver (`sql.Register(...)`).
- Registering with a global registry pattern that genuinely requires it (e.g., `image.RegisterFormat`).
- Setting up package-level constants that require computation.

These are zero-failure-mode registrations. Anything that *can fail* belongs in a constructor.

**How to spot.** `func init()` containing function calls beyond simple registrations or constant assignments. Calls to `os.Getenv` (config loading), `sql.Open` / DB connections, `http.Handle` (handler registration on `DefaultServeMux` — also a HTTP killer), file I/O, network I/O.

### I/O under lock (cross-reference)

Lives in `references/concurrency.md` as a concurrency-primitive issue, but is called out here because the *architectural* form — locking strategy that spans layers — is harder to catch than a single mutex misuse. When a service has a cache layer, a repository layer, and a transport layer, and the locking strategy bleeds across them, the I/O-under-lock anti-pattern becomes structural rather than local. The fix is layer redesign, not a one-line code change.

## Standard rubric

### Hidden global state

Package-level mutable variables (`var sessions = map[...]`) are global state masquerading as encapsulation. They couple every consumer of the package and resist testing in isolation. Senior reviewers flag any package-level `var` that is:

- Mutable.
- Read or written from more than one function.
- Initialized by an `init()` doing real work (see above).

Exception: package-level constants, sentinel errors, and registries that are *only* registered to (not read from) at runtime.

### Cyclic package dependencies

Go won't compile a true cycle, but the project structure often dances around one with shared "common" or "types" packages that import everything. Flag this pattern:

```
package handler  imports  package service
package service  imports  package model
package model    imports  package handler  ← red flag (or via "types")
```

The fix is usually moving the shared types to a package that no one else imports, or inverting a dependency via an interface defined where it's consumed.

### God packages

A package that does five unrelated things ("auth + user management + email + analytics + admin") is hard to test, hard to navigate, and slow to compile. Heuristic: if the public API has more than ~10 unrelated exported symbols, or the package depends on more than ~15 external packages, ask whether it should split.

### Transport / business logic mixing

HTTP handlers should be thin: parse → validate → call business logic → marshal response. Business logic should be testable without an HTTP request in scope.

```go
// Bad: business logic in the handler
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserReq
    json.NewDecoder(r.Body).Decode(&req)
    if !isValidEmail(req.Email) {
        // ... validation
    }
    user := User{Email: req.Email}
    h.db.Exec("INSERT INTO users ...")
    // ... 100 more lines
}

// Good: handler is thin
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserReq
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", 400); return
    }
    user, err := h.users.Create(r.Context(), req.ToInput())
    if err != nil {
        renderError(w, err); return
    }
    renderJSON(w, user)
}
```

The business logic lives in `users.Create`, which takes a domain input and returns a domain object — no `http.ResponseWriter` in sight.

### Repository leakage into handlers

If your handler calls `db.Query(...)` directly, your domain model has leaked. Indicators:

- Handler imports `database/sql` or your DB driver.
- Handler builds SQL strings.
- Handler knows about transactions.

The fix is a repository / data-access layer that exposes domain-shaped operations. The handler depends on an interface; the test substitutes a fake; the repository handles SQL.

### Context propagation across boundaries

`context.Context` should flow through every layer that does I/O or could be cancelled. If a function takes `ctx` and then calls a helper that doesn't, you've broken the chain — cancellation and tracing stop there. Flag any helper that does I/O or could block and doesn't take a context.

Exceptions: pure functions (no I/O, no blocking) don't need it. CPU-bound transformations don't need it unless they're long enough to want cancellation.

### Retry duplication across layers

If the HTTP client retries, the service layer retries, and the queue consumer retries, a single transient failure becomes 27 attempts (3 × 3 × 3). Retry logic belongs at exactly one layer — usually the boundary nearest the user-visible failure mode. Inner layers fail fast and propagate.

### Timeout ownership

Like retries, timeouts belong at one layer. The outermost timeout (request handler, RPC entry point) sets the budget; inner layers may *shorten* that budget but never extend it. Flag any inner function that constructs a fresh `context.WithTimeout` longer than the parent's remaining time.
