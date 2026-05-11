# Observability

Modern Go services live or die on observability. The difference between a five-minute outage and a five-hour one is usually whether the right log line / metric / span existed when the incident started. Senior review treats observability as a feature: missing observability on a critical path is MAJOR, not "nice to have."

## Structured logging over `fmt.Printf`

```go
// Bad: stringly-typed, ungreppable, fields can't be indexed
log.Printf("user %s failed to authenticate from %s: %v", userID, ip, err)

// Good: structured, machine-parseable, fields are first-class
logger.Error("authentication failed",
    "user_id", userID,
    "remote_ip", ip,
    "error", err,
)
```

Standard library: `log/slog` (Go 1.21+). Third-party: `zap`, `zerolog`. Pick one per service; don't mix.

Flag `fmt.Printf`, `log.Printf`, `log.Println` in production code as MINOR (or MAJOR if it's on a frequently-hit path where the lack of structure makes incident response harder).

## No secrets, tokens, or PII in logs

This is BLOCKER for any field that contains a secret. Common offenders:

- Authorization headers: `Bearer <token>`.
- Session cookies.
- API keys.
- Password fields in form bodies.
- DSN strings with embedded passwords.
- Full request bodies on auth endpoints.
- User PII subject to compliance (GDPR, HIPAA, etc.) — email, phone, name, address depending on jurisdiction.

Logging the request method, URL path (no query string for sensitive endpoints), response status, and duration is almost always safe. Anything beyond that needs a deliberate decision about sensitivity.

```go
// Bad
logger.Info("request", "headers", r.Header) // ← Authorization in there

// Good
logger.Info("request",
    "method", r.Method,
    "path", r.URL.Path,
    "user_id", auth.UserID(ctx), // identity from validated session, not raw token
)
```

## Request and trace IDs propagated through context

Every request that enters the service should have a unique ID, attached to the context, and included in every log line and outbound call:

```go
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = newRequestID()
        }
        ctx := context.WithValue(r.Context(), reqIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

For distributed tracing, propagate the trace ID + span ID via context (OpenTelemetry SDK handles this if you use it). The trace ID lets you stitch logs across services for one request.

## Metrics around retries, failures, timeouts

Counters alone are not enough; you need histograms for latency and counters bucketed by outcome:

```go
// At each retry attempt
retriesTotal.WithLabelValues("upstream_x", "rate_limited").Inc()

// On final success/failure
requestsTotal.WithLabelValues("upstream_x", "ok"|"error").Inc()
requestDuration.WithLabelValues("upstream_x").Observe(duration.Seconds())
```

If the code has a retry loop without metrics on retry attempts, an outage where every request retries silently and succeeds eventually is invisible — until a downstream rate limit pushes it over.

## Error causality preserved (`%w`)

See `references/errors.md`. The observability angle: if errors are wrapped with `%w` all the way up, a single log line at the top of the stack contains the full chain — `"handling request: querying user: dial postgres: connection refused"` — which is all an on-call needs. If errors are flattened with `%v` partway up, the log line at the top has lost the inner detail.

## No duplicate logging across layers

A common anti-pattern: every layer logs the error before returning it.

```go
// Bad
func (s *Service) Foo(ctx context.Context) error {
    if err := s.repo.Foo(ctx); err != nil {
        log.Error("repo failed", "error", err)   // log here
        return fmt.Errorf("service Foo: %w", err) // ...and return
    }
    return nil
}
```

The handler above this also logs. The middleware above the handler also logs. One error becomes three log lines, with the inner two missing the request context.

**Rule:** the layer that *handles* the error (decides what to do about it: retry, return 500, propagate to user) logs. Every other layer just wraps and propagates. Usually that means the HTTP middleware logs (with full request context), and the inner layers stay quiet.

## Span / trace boundaries

If the service uses distributed tracing, every meaningful operation should start a span:

```go
ctx, span := tracer.Start(ctx, "UserService.Create")
defer span.End()

user, err := s.repo.Create(ctx, input)
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
    return nil, err
}
```

The hierarchy matters: spans nest by parent-child via context. A service call that doesn't accept ctx breaks the chain — and breaks the trace. (See `references/architecture.md` on ctx propagation.)

## Log levels and rates

- **`debug`**: development. Should be off in production by default, toggle-able.
- **`info`**: normal operation. Per-request logs at info are fine for low-traffic services; for high-traffic services they overwhelm log storage — sample.
- **`warn`**: recoverable degradation (a retry succeeded after failures, a fallback was used). These should be rare and meaningful.
- **`error`**: unrecoverable failures that an on-call would want to know about. Page-able.

Flag high-frequency code paths logging at `info` per request as MINOR (cost of logs at scale). Flag suspicious `error`-level logs that fire on normal-but-rare events as MAJOR (alert fatigue).

## Health endpoints

`/healthz` (or `/readyz`) is a standard, but it's often misimplemented:

- `/healthz` should be **fast** and **stateless** — Kubernetes probes hit it on every liveness check.
- It should *not* deeply check every dependency on every call. That cascades probe failures.
- A separate readiness probe (`/readyz`) can do deeper checks if needed, and only at deploy/rollout time.

## Avoid logging the same context twice

The middleware logs `method=GET path=/api/foo`. The handler then logs `"handling foo"`. The next call logs `"calling subsystem"`. Each log line repeats most of what was in the previous. This costs storage and obscures the actually-interesting fields.

Either: emit one log line per request (with all context) at the boundary, or use structured logging context with `slog.With` so child loggers inherit fields without restating them.

## Stack traces

`debug.Stack()` is appropriate for unexpected panics and for failures that benefit from a one-shot capture. Don't emit a stack trace per error in normal operation — they're expensive and most of the time uninformative for a well-`%w`-chained error.
