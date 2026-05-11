# Security

Go-specific security review. This file does not cover all of application security — that's a broader topic best served by `/security-review`. This file focuses on the Go-language-specific pitfalls: places where Go's standard library, idioms, or API shapes make insecure code easy to write.

## SQL injection

Parameterized queries are mandatory for any value that originated outside the code:

```go
// Bad
db.QueryContext(ctx, fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name))

// Good
db.QueryContext(ctx, "SELECT * FROM users WHERE name = $1", name)
```

This is non-negotiable. Any string concatenation of user input into a SQL query is BLOCKER. Even values "from internal services" or "validated upstream" — defense in depth, no exceptions.

The narrow exception: dynamic identifiers (table names, column names) that *cannot* be parameterized by `database/sql`. For these, validate against a strict allowlist before interpolating. Reject anything not in the allowlist.

## `crypto/rand` vs `math/rand`

`math/rand` is a deterministic pseudo-random generator. Acceptable for: simulations, shuffles in non-security contexts, jitter for retries. Never acceptable for: tokens, session IDs, password reset codes, nonces, key material, CSRF tokens, or anything an attacker should not be able to guess.

```go
import (
    mathrand "math/rand"
    cryptorand "crypto/rand"
)

// Bad: predictable token
token := strconv.Itoa(mathrand.Int())

// Good: cryptographically random
b := make([]byte, 32)
if _, err := cryptorand.Read(b); err != nil {
    return "", err
}
token := base64.RawURLEncoding.EncodeToString(b)
```

Note: `math/rand/v2` (Go 1.22+) is still not cryptographically secure despite the v2. Always use `crypto/rand` for security-sensitive randomness.

## Constant-time comparison

Comparing secrets (tokens, MACs, password hashes) with `==` is a timing attack vector — the comparison short-circuits at the first byte mismatch, leaking byte-by-byte equality timing.

```go
// Bad
if userToken == expectedToken { ... }

// Good
if subtle.ConstantTimeCompare([]byte(userToken), []byte(expectedToken)) == 1 { ... }

// Good for HMACs specifically
if hmac.Equal(receivedMAC, computedMAC) { ... }
```

Required for: HMAC signatures, session tokens, API keys, CSRF tokens, anything where unauthorized access is the threat model.

## Path traversal

User-supplied filenames or path components can escape the intended directory via `..` or absolute paths:

```go
// Bad: attacker passes "../../etc/passwd"
file, _ := os.Open(filepath.Join(uploadDir, userFilename))

// Good: clean + verify
cleaned := filepath.Clean(filepath.Join(uploadDir, userFilename))
abs, err := filepath.Abs(cleaned)
if err != nil { return err }
absUploadDir, _ := filepath.Abs(uploadDir)
if !strings.HasPrefix(abs, absUploadDir+string(os.PathSeparator)) {
    return errors.New("path traversal blocked")
}
file, _ := os.Open(abs)
```

`filepath.Clean` alone is **not** sufficient — it canonicalizes `..` but doesn't prevent escaping a base directory. The boundary check is the security control.

Go 1.20+ has `os.Root` and `os.OpenInRoot` which provide a safer API for this exact case.

## `http.MaxBytesReader` on untrusted bodies

Without it, an attacker streams a 10 GB request body and exhausts memory. See `references/http-and-resources.md`.

## Default `http.Client` Timeout

`http.DefaultClient` has no timeout. See `references/http-and-resources.md` Production-killer patterns. From a security angle: a hung outbound call from your service to a third party is a slow-loris-style DoS against your own service.

## TLS configuration

Default `tls.Config{}` accepts TLS 1.0 in some contexts. For production servers and clients:

```go
&tls.Config{
    MinVersion: tls.VersionTLS12, // or TLS13
    // CipherSuites: leave nil to use Go's secure defaults (Go 1.17+)
}
```

Flag any `tls.Config` that:
- Sets `InsecureSkipVerify: true` outside of clearly-marked test code.
- Doesn't set `MinVersion`.
- Manually configures cipher suites with weak entries (RC4, 3DES, anything with `_SHA` only).

## Subprocess execution

`exec.Command(name, args...)` does NOT invoke a shell. The arguments are passed as-is. That makes it safer than the shell-string form, but only if the arguments themselves don't contain control characters that the *target program* interprets.

Specifically: never pass user-controlled strings to programs that themselves parse shell metacharacters (`sh -c`, `bash -c`, `eval`-style invocations). And never use `exec.Command("sh", "-c", userString)` — that defeats the safer API entirely.

## Secrets in logs

Logging request bodies, headers (especially `Authorization`, `Cookie`), or response bodies risks leaking secrets to log aggregators, third-party log services, and operator screens. See `references/observability.md` for the structured-logging rules. Security flag here: any `log.Printf("%+v", req)` or equivalent on a request with sensitive fields.

## Secrets in error messages

```go
// Bad: leaks DSN to whoever sees the error
return fmt.Errorf("connecting to %s: %w", dsn, err)
```

The DSN includes the password. The error propagates up the call stack and may be logged, returned in an HTTP response, or sent to an error tracker. Sanitize before formatting.

## File permissions

`os.WriteFile(path, data, perm)` — the `perm` parameter is applied with the process umask. For files that may contain secrets (credentials, tokens, private keys):

```go
os.WriteFile(path, data, 0600) // owner read/write only
```

Flag `0644` or `0666` on files holding secrets.

## `unsafe` package

Any use of `unsafe.Pointer`, `unsafe.Slice`, `unsafe.String`, or `unsafe.SliceData` is a potential memory-safety issue. Acceptable in:

- Performance-critical conversions where allocation cost has been measured.
- Interop with C or syscall data structures.
- Standard library internals.

Not acceptable as "I wanted to be clever." Flag any new `unsafe` usage in application code as MAJOR; require justification.

## `os.Getenv` for secrets in code

Reading secrets from environment variables is fine. But:

```go
apiKey := os.Getenv("API_KEY") // ok
// ...
log.Info("starting", "api_key", apiKey) // ← logs the secret
```

Audit the lifecycle: secret read → never logged, never returned in an error, never echoed in a debug endpoint.

## CSRF / XSS / open redirect

These are application-level concerns, not Go-language pitfalls, but flag obvious cases:
- HTML templates using `template/html` (auto-escaping) vs `text/template` (no escaping) for HTML output.
- `http.Redirect(w, r, userSuppliedURL, ...)` without a same-host check.
- Missing CSRF protection on state-changing endpoints from cookie-authenticated clients.
