# HTTP & resources

HTTP clients, HTTP servers, and resource hygiene (bodies, connections, files, transactions). This category is where most Go services accumulate slow-burn production issues — bugs that pass tests, slip into production, and surface as latency spikes a week later under load.

## Production-killer patterns

### `http.Get` / `http.Post` / `http.DefaultClient`

**Why this matters in production.** `http.Get`, `http.Post`, `http.PostForm`, and any other `http.Default*` family member uses `http.DefaultClient`, which has **no timeout** of any kind. A single slow or hung upstream stalls the calling goroutine forever. Pile up a few of these on a busy handler and you exhaust goroutines, exhaust the connection pool, and the service stops responding. The default client also has no retry budget, no circuit breaker, and no observability — there is no situation in production code where it is the right answer.

**Bad:**
```go
resp, err := http.Get(url)
```

**Good:**
```go
var client = &http.Client{
    Timeout: 10 * time.Second,
    // Transport with explicit timeouts when needed
}

resp, err := client.Get(url)
```

**How to spot.** Search the diff for `http.Get(`, `http.Post(`, `http.PostForm(`, `http.Head(`, `http.DefaultClient`, `http.DefaultTransport`. All are red flags in non-test production code. In tests, `http.DefaultClient` is sometimes acceptable against a `httptest.Server`, but even there an explicit client is clearer.

### `http.DefaultServeMux`

**Why this matters in production.** `http.DefaultServeMux` is a process-global. Any imported package can register routes on it without the importer's knowledge. The most common case is `net/http/pprof`, which on import registers `/debug/pprof/*` handlers on `DefaultServeMux`. If your server uses `http.ListenAndServe(":80", nil)` — i.e., uses `DefaultServeMux` — your runtime profiling and goroutine dumps are exposed on the public listener. That's a confidentiality leak at best (internal architecture exposed) and a DoS vector at worst (`/debug/pprof/profile` runs a 30-second CPU profile that pegs a core).

**Bad:**
```go
import _ "net/http/pprof" // registers on DefaultServeMux
// ...
http.HandleFunc("/api", apiHandler) // also on DefaultServeMux
http.ListenAndServe(":8080", nil)   // uses DefaultServeMux
```

**Good:**
```go
import _ "net/http/pprof" // still registers on DefaultServeMux, but...
// ...
mux := http.NewServeMux()
mux.HandleFunc("/api", apiHandler)
srv := &http.Server{Addr: ":8080", Handler: mux, ...}
srv.ListenAndServe()

// Bind pprof to a separate, internal-only listener:
go http.ListenAndServe("127.0.0.1:6060", http.DefaultServeMux)
```

**How to spot.** `http.ListenAndServe(addr, nil)`, `http.Handle(...)` or `http.HandleFunc(...)` at the package level, or any code that doesn't pass a `Handler` to `http.Server`. Also flag `import _ "net/http/pprof"` alongside a public listener.

## Standard rubric

### `http.Client` configuration

- **`Timeout`** is mandatory. Without it, requests can hang indefinitely. A reasonable default is 10–30s; tune per-endpoint as needed.
- For finer control, configure `Transport` with `DialContext`, `TLSHandshakeTimeout`, `ResponseHeaderTimeout`, `ExpectContinueTimeout`. The single `Client.Timeout` covers the whole exchange end-to-end and is usually what you want.
- Reuse one `Client` across requests. Don't construct a new client per call — you lose connection pooling and pay TLS handshake on every call.

### `http.Server` configuration

The server side needs explicit timeouts to defend against slowloris and client misbehavior:

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,   // CRITICAL: defends against slowloris
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      60 * time.Second,
    IdleTimeout:       120 * time.Second,
}
```

`ReadHeaderTimeout` in particular is the cheapest way to neutralize the slowloris attack class. Treat its absence on a publicly-accessible server as BLOCKER.

### `defer Close()` after the error check

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close() // ← AFTER the error check, not before
```

`defer resp.Body.Close()` before the error check panics with a nil-pointer dereference when `client.Get` returns an error and a nil response. Same pattern for `os.Open`, `sql.DB.Query`, etc.

### `http.MaxBytesReader` on untrusted request bodies

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MiB cap
```

Without it, an attacker can stream a gigabyte body and exhaust memory or stall the handler. Required for any handler that reads an untrusted body.

### Drain the response body before close on errors

The Go HTTP client only reuses connections if the response body is fully read and closed. On error paths it's tempting to `defer body.Close()` and return immediately, but if the body has unread bytes you've just disabled connection reuse for that connection. For non-trivial response bodies on the error path:

```go
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

### Retry safety on non-idempotent methods

`GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`, `TRACE` are idempotent and safe to retry on transport errors. `POST` and `PATCH` are not — retrying without an idempotency key risks double-billing, double-publishing, duplicate side effects. Reviewer asks: does this retry policy distinguish methods?

### Middleware ordering

Standard order, outer to inner:

1. **Recovery** — catches panics so they don't kill the server.
2. **Logging** — even if everything else fails, the request is logged.
3. **Tracing / request ID** — every subsequent layer benefits from the trace context.
4. **Metrics** — measures the whole pipeline including auth.
5. **Auth / authz** — rejects before doing real work.
6. **Rate limiting** — close to the handler to limit by authenticated identity.
7. **Handler.**

Recovery before logging means panics still get logged. Logging before auth means failed auth attempts are visible. Reviewer flags inverted orderings.

### Panic recovery boundaries

Every HTTP handler chain needs a recovery middleware at the top. Background goroutines need explicit `recover()` at the goroutine boundary — otherwise an uncaught panic kills the entire process.

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Error("worker panic", "panic", r, "stack", debug.Stack())
        }
    }()
    work()
}()
```

### Graceful shutdown

`srv.Shutdown(ctx)` stops accepting new connections and waits for in-flight requests up to the context deadline. Required for any long-lived server. Without it, deploys cut active connections mid-flight, returning errors to users instead of just shifting traffic.

### `io.ReadAll` only on bounded readers

`io.ReadAll(r)` reads everything until EOF or error. On an untrusted reader (HTTP request body, stdin from a subprocess, file from user-supplied path) this is a memory exhaustion vector. Pair every `io.ReadAll` with a bounded reader (`io.LimitReader`, `http.MaxBytesReader`, or an explicit size check before reading).
