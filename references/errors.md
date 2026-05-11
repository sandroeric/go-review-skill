# Errors

Error handling in Go is explicit and verbose by design. The verbosity is a feature: every error has a visible disposition. Senior review focuses on whether that disposition is correct.

## Production-killer patterns

### Error masking in `defer Close()` on writers

**Why this matters in production.** When you write to a file, the OS buffers the writes. The actual flush to disk happens on `Close()`. If the disk is full, the storage is unmounted, or the underlying device errors out, `Write()` will often succeed (because it just buffered) while `Close()` returns the real failure. Ignoring `Close()`'s error gives you silent data loss — the user thinks the export, the upload, the log, the snapshot succeeded; in reality it never hit stable storage.

**Bad:**
```go
func writeReport(path string, data []byte) error {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer f.Close() // ← error swallowed; silent data loss on flush failure
    _, err = f.Write(data)
    return err
}
```

**Good:**
```go
func writeReport(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        err = errors.Join(err, f.Close())
    }()
    _, err = f.Write(data)
    return err
}
```

**How to spot.** `defer f.Close()` (or any `Close()`/`Sync()` on a writer/encoder/transactor) without a named return value capturing the close error. Applies to: `os.File` when writing, `bufio.Writer` (use `Flush` then `Close`), `gzip.Writer`, `csv.Writer`, `*sql.Tx` (commit/rollback), `*sql.Stmt`, network connections where buffered writes matter.

**Note.** Read-only consumers (`http.Response.Body`, `*sql.Rows`, opened-for-read files) are different — the worst case of ignoring `Close()` there is a leaked descriptor, not data loss. Still want to close them, but `defer body.Close()` without error check is fine for reads.

## Standard rubric

### `%w` wrapping vs `%v` logging

- `fmt.Errorf("loading config: %w", err)` — preserves the chain; callers can `errors.Is` / `errors.As` against it.
- `fmt.Errorf("loading config: %v", err)` — flattens to a string. Only acceptable at terminal logging sites where no caller will inspect the error.
- Mixing both in the same chain is a smell — picks one shape, sticks with it.

### `errors.Is` and `errors.As`

```go
if errors.Is(err, sql.ErrNoRows) { ... }            // sentinel check, walks the chain
var pathErr *os.PathError
if errors.As(err, &pathErr) { ... }                 // typed extraction
```

Flag `strings.Contains(err.Error(), "no rows")` or `err == sql.ErrNoRows` as MAJOR — the first is fragile, the second misses wrapped errors.

### Silent swallow

```go
_ = doThing() // ← red flag
```

Acceptable only with a comment explaining why the error is ignorable in this exact context (e.g., "best-effort cleanup; primary path already failed"). Without a comment, treat as MINOR.

### Double-handling

```go
if err != nil {
    log.Printf("failed: %v", err)  // log
    return err                     // and return
}
```

This logs the error at every layer the call passes through. The caller logs it too, and the caller's caller. You end up with the same error in the log 5 times with no useful added context.

**Rule:** log *or* return, not both. The layer that handles the error logs it. Every layer above that just propagates with added context via `%w`.

### Sentinel errors

```go
var ErrNotFound = errors.New("not found")
```

Expose a sentinel only when callers genuinely need to branch on it. If no caller actually checks `errors.Is(err, pkg.ErrSomething)`, the sentinel adds API surface for no value. Custom error types with behavior (`Timeout() bool`, etc.) are similarly only justified when callers use them.

### Adding context up the stack

The caller should know *what failed* from the error string alone. `"connection refused"` is useless; `"loading user config from /etc/app/config.yaml: open /etc/app/config.yaml: connection refused"` lets ops figure it out without reading code.

Conventional shape: `verb-phrase: %w` where the verb describes what *this* function was trying to do. Don't include the file/line — `%w` already preserves the chain.

### When to `panic`

Only in `main`/`init` for unrecoverable startup failures, or for programmer errors that should never occur at runtime (invariant violations). Never as control flow. Never in library code that callers can't recover from.

Flag any `panic(err)` in non-startup, non-test code as MAJOR.
