# Concurrency

Goroutines, channels, context, mutexes, timers — the surface area where Go shines and where Go services die under load. Most production outages in a Go codebase trace back to one of these patterns. Apply the rubric below carefully.

## Production-killer patterns

These pass `go vet`, `staticcheck`, and standard CI but cause real production outages. Default to **BLOCKER** or **MAJOR** severity at **HIGH** confidence unless surrounding context proves the concern is neutralized.

### `time.After` inside a `for/select` loop

**Why this matters in production.** `time.After(d)` allocates a new `Timer` and the runtime cannot garbage-collect it until it fires. In a tight `for/select` with another active channel, the timer is replaced every iteration but the previous one keeps living in the runtime's heap until the original duration elapses. Under sustained throughput this leaks tens of thousands of pending timers; memory pressure climbs, the runtime spends increasing time managing the timer heap, latency degrades, and the service eventually OOMs.

**Bad:**
```go
for {
    select {
    case msg := <-ch:
        handle(msg)
    case <-time.After(5 * time.Second):
        keepalive()
    }
}
```

**Good:**
```go
t := time.NewTimer(5 * time.Second)
defer t.Stop()
for {
    select {
    case msg := <-ch:
        handle(msg)
        if !t.Stop() {
            <-t.C
        }
        t.Reset(5 * time.Second)
    case <-t.C:
        keepalive()
        t.Reset(5 * time.Second)
    }
}
```

**How to spot.** Search the diff for `time.After(` inside any `for` body or long-lived `select`. One-shot `time.After` outside a loop is fine.

### Context detachment for background work from request handlers

**Why this matters in production.** Spawning a goroutine inside an HTTP handler that uses the request's `ctx` means the background work is cancelled as soon as the user disconnects (closed tab, mobile network change, client timeout). The result is incomplete database writes, dropped analytics, half-applied state changes — and the symptoms are intermittent and impossible to reproduce locally.

**Bad:**
```go
func handler(w http.ResponseWriter, r *http.Request) {
    go publishEvent(r.Context(), event) // dies if client disconnects
    w.WriteHeader(http.StatusAccepted)
}
```

**Good (Go 1.21+):**
```go
func handler(w http.ResponseWriter, r *http.Request) {
    bgCtx := context.WithoutCancel(r.Context()) // preserves trace IDs, sheds cancellation
    go publishEvent(bgCtx, event)
    w.WriteHeader(http.StatusAccepted)
}
```

**Good (pre-1.21):** detach manually — pass `context.Background()` and re-attach only the values you need.

**How to spot.** Any `go func(...)` inside an HTTP handler, gRPC handler, or other request-scoped function where the goroutine closes over the request `ctx` and outlives the handler return.

### I/O under lock

**Why this matters in production.** Holding a `sync.Mutex` or `sync.RWMutex` while calling out to the network or a database means the goroutine sits idle on network latency *with the lock held*. Every other goroutine waiting for that lock piles up. One slow upstream → entire process stalls → cascading failure across the service mesh.

**Bad:**
```go
func (c *Cache) RefreshUser(id string) error {
    c.mu.Lock()
    defer c.mu.Unlock()
    user, err := c.db.QueryUser(id) // ← network call under lock
    if err != nil {
        return err
    }
    c.users[id] = user
    return nil
}
```

**Good:**
```go
func (c *Cache) RefreshUser(id string) error {
    user, err := c.db.QueryUser(id) // I/O outside the lock
    if err != nil {
        return err
    }
    c.mu.Lock()
    c.users[id] = user
    c.mu.Unlock()
    return nil
}
```

**How to spot.** Any `mu.Lock()` / `defer mu.Unlock()` block that contains a function call to anything network-shaped: DB drivers, HTTP clients, gRPC stubs, message queue producers, file I/O on slow storage.

### Unbounded goroutine fan-out over untrusted input

**Why this matters in production.** `for _, item := range userInput { go process(item) }` is a denial-of-service vector. An attacker (or a misbehaving upstream) sends a million items and you spawn a million goroutines. Memory exhausts, the scheduler thrashes, the service dies.

**Bad:**
```go
for _, id := range req.IDs {
    go fetchAndStore(id)
}
```

**Good:**
```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(16)
for _, id := range req.IDs {
    g.Go(func() error {
        return fetchAndStore(ctx, id)
    })
}
if err := g.Wait(); err != nil {
    return err
}
```

**How to spot.** Loops with `go func()` over slices that derive from user input (request bodies, query params, message payloads, file reads). Always require explicit concurrency bounding.

## Standard rubric

### Goroutine lifecycle

Every `go func()` must have a clear exit path. Reviewer asks: under what condition does this goroutine return? If the answer is "when the process dies," that's a leak. Acceptable exits:

- Context cancellation (`<-ctx.Done()`).
- Closed channel (`for msg := range ch`).
- Finite work that completes.

### Context plumbing

- `ctx context.Context` is the **first parameter**, always.
- Never stored in a struct field.
- Never `context.TODO()` outside of tests or stubs being filled in.
- Long-running background work that doesn't have a parent context: start from `context.Background()` and attach cancellation/deadlines deliberately.

### Channel ownership

- The sender closes. Always.
- Receivers must handle closed channels gracefully (the second return value of `<-ch`).
- Multi-sender requires explicit coordination — usually a separate "done" channel or a `sync.Once` guard around `close`.

### Mutex hygiene

- Smallest possible scope.
- `defer mu.Unlock()` immediately after `mu.Lock()` (or be very explicit about why not).
- No I/O under lock (see Production-killer patterns).
- Prefer `sync.RWMutex` only when reads vastly outnumber writes and contention is measured.

### `sync.WaitGroup` placement

- `wg.Add(n)` happens **before** the corresponding `go`, in the caller's stack. Never inside the goroutine.
- `wg.Done()` is deferred at the top of the goroutine body.
- `wg.Wait()` is called after every `Add`.

### Loop-variable capture (Go-version-gated)

**Go <1.22:** classic capture bug.
```go
for _, v := range items {
    go func() { use(v) }() // all goroutines see the last v
}
```
Flag as BLOCKER. Fix: `go func(v string) { use(v) }(v)` or `v := v` before the goroutine.

**Go ≥1.22:** loop variables are per-iteration. Only flag captures that involve pointers or shared mutation across iterations — those are still hazards regardless of version. Don't flag plain value captures on Go 1.22+.

### `errgroup` patterns

- `errgroup.WithContext` for fan-out where any failure should cancel the rest.
- `errgroup.SetLimit(n)` to bound concurrency. Without it, `errgroup` does not limit goroutines.
- `g.Wait()` returns the first non-nil error and waits for all goroutines.

### Ticker and Timer cleanup

- `time.NewTicker` → `defer ticker.Stop()`.
- `time.NewTimer` → `defer t.Stop()`. If you read from `t.C` or call `t.Reset`, follow the `Stop`/`drain` pattern.
- See Production-killer patterns for `time.After` in loops.

### `select` patterns

- A `select` with only a `default` branch is a busy poll. Almost always wrong.
- A `select` without a `default` blocks until one case is ready — that's usually what you want.
- `select` over many channels can starve some cases under load; consider explicit fairness or a different shape.

### `context.WithCancelCause`

Use when downstream code needs to distinguish *why* the context was cancelled (timeout vs. shutdown vs. user-initiated abort). `context.Cause(ctx)` returns the original cause; the standard `ctx.Err()` returns the generic `context.Canceled`.
