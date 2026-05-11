# Performance

Go performance review is mostly an exercise in restraint. The compiler and runtime are good. Most "optimizations" don't matter, and many actively hurt readability or correctness. A senior reviewer flags performance issues only when they have real production impact.

## When to flag a performance issue

A performance finding requires **at least one** of:

- Code on a request-path handler (per-request hot path).
- Batch processing or stream processing over large inputs.
- A loop over unbounded input (anything where N is set by user data).
- An allocation visible in profiling output the user has shared.
- An existing benchmark in the test suite that demonstrates the problem.

If none of the above applies, downgrade to NIT or drop the finding entirely. "This could allocate slightly less" in a one-shot init function is not worth a code review comment.

## Production-killer patterns

### Pointer-default for returns and receivers without need

**Why this matters in production.** Returning a pointer (`*MyStruct`) from a function is often slower than returning a value, despite the folk wisdom that "passing values is slow." The reason: returning a pointer to a local variable forces the compiler's escape analysis to allocate the struct on the heap. At high throughput, the GC pressure from millions of tiny heap allocations dominates the cost of copying a 40-byte struct on the stack. P99 latency spikes correlate with GC pauses, not with the cost of `mov` instructions.

**Bad (in a hot path):**
```go
func makeUser(id string) *User {
    return &User{ID: id, ...} // escapes to heap
}
```

**Good:**
```go
func makeUser(id string) User {
    return User{ID: id, ...} // stays on stack at the caller
}
```

**When pointers are still correct:**
- The struct is large (>~128 bytes, measure to be sure).
- The caller needs to mutate the returned value.
- The type contains `sync.Mutex`, `sync.WaitGroup`, or other no-copy types.
- It represents a singleton or shared identity.

**How to spot.** Constructors / factory functions returning `*T` where the caller doesn't mutate the result. Functions in request-path code that return pointers to local-only structs. Verify with `go build -gcflags=-m` to see escape analysis output.

### Interface boxing in hot loops

**Why this matters in production.** Assigning a concrete value to an `interface{}` / `any` parameter or return type requires the runtime to allocate a wrapper (the "iface" structure with a type pointer and a data pointer). On the hot path, every call thrashes the garbage collector — you may see allocs/op climb into the millions even for code that looks like it should be zero-alloc.

**Bad:**
```go
func sum(xs []any) any {        // boxes every element
    var total int
    for _, x := range xs {
        total += x.(int)         // unbox on every iter
    }
    return total                 // box the result
}
```

**Good (generics, Go 1.18+):**
```go
func sum[T int | float64](xs []T) T {
    var total T
    for _, x := range xs {
        total += x
    }
    return total
}
```

**Good (concrete type when polymorphism isn't actually needed):**
```go
func sumInts(xs []int) int {
    var total int
    for _, x := range xs {
        total += x
    }
    return total
}
```

**How to spot.** Hot-path functions with `any` / `interface{}` parameters or returns. Loops that call `interface{}`-shaped APIs millions of times per request.

## Standard rubric

### Slice and map pre-sizing

When you know the final size (or have a tight upper bound), pre-size:

```go
out := make([]Result, 0, len(input)) // ← capacity hint
for _, x := range input {
    out = append(out, transform(x))
}

m := make(map[string]int, expectedCount)
```

Without the capacity hint, append grows the backing array geometrically (multiple allocations + copies). Pre-sizing is a one-line change with measurable benefit on hot paths.

### `strings.Builder` for string concatenation in loops

```go
// Bad: O(n²) — each += allocates a new string
result := ""
for _, s := range parts {
    result += s
}

// Good: O(n)
var b strings.Builder
b.Grow(estimatedSize) // optional, helps if you can estimate
for _, s := range parts {
    b.WriteString(s)
}
result := b.String()
```

For two or three concatenations outside a loop, `+` is fine and reads more naturally.

### `[]byte` ↔ `string` conversions

`[]byte(s)` and `string(b)` allocate and copy. In a hot path, watch for:

- Converting a `[]byte` to `string` just to compare against a literal — `bytes.Equal([]byte("foo"), b)` avoids the alloc.
- Map keys: `m[string(b)]` is special-cased by the compiler to avoid the allocation when used as a lookup. Still don't store the key as `string` if you can keep `[]byte`.
- `unsafe.String` / `unsafe.SliceData` (Go 1.20+) can avoid the copy but defeats the borrow-checker — only with measured benefit and careful lifetime reasoning.

### Receiver consistency

A type's method set should use a consistent receiver kind — all value, or all pointer. Mixing them is confusing and causes subtle issues with interface satisfaction. Default to pointer receivers for types with any mutating methods, value receivers for purely read-only types (rare).

### Reflection in hot paths

`reflect` is expensive — runtime type checks, allocations, type-switch overhead. Acceptable in startup code (config decoding, route registration). Flagged in per-request code. Common culprits: JSON encoding/decoding of large objects, generic transformers, type-switch-heavy dispatch.

Generics (Go 1.18+) replace many reflection-based patterns with compile-time-monomorphized code.

### `sync.Pool` pitfalls

`sync.Pool` reduces allocation pressure for short-lived objects that are constructed identically. Common pitfalls:

- Putting pointers to objects that retain references to large memory (leaks via the pool).
- Forgetting to reset the object before `Put` (next user sees stale data).
- Using a pool where the allocation cost is negligible (premature complexity).

`sync.Pool` is appropriate when profiling shows allocation pressure on a specific type. Not appropriate as a default tool.

### Profile first

Performance findings without profile evidence are speculation. If the user hasn't profiled, the right move is usually to point at the suspicious pattern with **LOW** confidence and suggest `go test -bench` or `pprof`. Don't promote speculation to MAJOR/BLOCKER.
