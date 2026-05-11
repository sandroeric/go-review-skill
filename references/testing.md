# Testing

Test quality is the most reliable proxy for code quality. A change with weak tests is more likely to regress, more likely to hide a design problem, and more expensive to refactor later. Senior review treats tests as first-class code.

## Table-driven with `t.Run` subtests

The canonical shape for testing a function across multiple inputs:

```go
func TestParseUser(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    User
        wantErr bool
    }{
        {"empty", "", User{}, true},
        {"valid", `{"id":"u1"}`, User{ID: "u1"}, false},
        {"malformed json", `{`, User{}, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseUser(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("got err=%v, wantErr=%v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
```

Benefits: each case is named (test failure messages identify the case), subtests can be run individually (`go test -run TestParseUser/valid`), and adding a new case is a one-line edit.

Flag non-table tests with 3+ similar test functions as MINOR.

## `t.Parallel()` where safe

`t.Parallel()` makes the test run in parallel with other parallel tests in the package. Safe when:

- The test does not modify shared state (env vars, working directory, global vars).
- The test does not depend on an external service whose contract is not concurrent-safe.

Within a table-driven test:

```go
for _, tt := range tests {
    tt := tt // capture (only needed on Go <1.22)
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

Flag missing `t.Parallel()` on table-driven tests as MINOR. Flag misuse (using `t.Parallel()` with `t.Setenv` or shared mutable state) as MAJOR.

## `t.Helper()` on assertion helpers

A helper function that calls `t.Errorf` or `t.Fatalf` should call `t.Helper()` first. Without it, test failures point to the helper's source line instead of the caller's — useless for diagnosing which subtest failed.

```go
func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}
```

## Error paths tested, not just happy paths

The most common test-quality failure is testing only the success case. A change that adds an error branch should add a test that exercises that branch.

For each `if err != nil { return ... }` or other failure path:

- Is there a test that produces this error?
- Does the test assert on the error's identity (`errors.Is`, `errors.As`) rather than just "an error occurred"?
- For BLOCKER/MAJOR error paths (e.g., DB transaction rollback), the test is non-negotiable.

## `testdata/` for golden files

Complex outputs (generated SQL, JSON, HTML, code generation) belong in `testdata/`. The test reads the expected output from a file and compares.

```go
golden := filepath.Join("testdata", t.Name()+".golden")
got := generate(input)
want, _ := os.ReadFile(golden)
if !bytes.Equal(got, want) {
    if *update { // a -update flag
        os.WriteFile(golden, got, 0644)
        return
    }
    t.Errorf("output differs from %s", golden)
}
```

The `-update` flag pattern lets the developer regenerate goldens after intentional changes, with a diff visible in the commit.

## Race tests on concurrent code

Any code that uses goroutines, channels, mutexes, or atomic operations should have a test that's executed under the race detector. Add `t.Parallel()` *and* exercise the concurrent paths from multiple goroutines within the test.

The user can run `go test -race ./...` to surface races. The skill itself doesn't run race tests by default — but the reviewer flags concurrent code that has no concurrent-shaped test as MAJOR.

## Benchmark hygiene

```go
func BenchmarkParse(b *testing.B) {
    input := loadFixture("large.json")
    b.ResetTimer()        // exclude setup time
    b.ReportAllocs()      // report allocations per op
    for i := 0; i < b.N; i++ {
        _, _ = Parse(input)
    }
}
```

Common mistakes:
- Allocating the input inside the loop (measuring allocation, not the function).
- Not calling `b.ResetTimer` after setup.
- Comparing a benchmark across machines / kernels without recognizing the noise floor.

## Meaningful test names

`TestFoo`, `TestParse`, `TestStuff` are useless when one of them fails. Names should communicate what's being tested *and* the expected outcome.

Common patterns:
- `TestX_WhenY_ThenZ` — explicit, longest, clearest.
- `TestX_HappyPath` / `TestX_ErrorOnY` — when X is the SUT and the case is short.
- Subtests via `t.Run("when input is empty", ...)` for the per-case naming.

Avoid generic suffixes like `_Test`, `_Basic`, `_OK`.

## Real vs. fake dependencies

When a function takes `*sql.DB` or an `http.Client`, the test has two options:

1. **Real (integration test).** Spin up an actual Postgres / `httptest.Server`. Slow, accurate, catches integration bugs. Use when the contract with the dependency is non-trivial.
2. **Fake (unit test).** Pass an interface satisfied by a small test double. Fast, isolates the function. Use when the function's logic is the interesting thing and the dependency is straightforward.

Senior judgment: a service that touches a database extensively should have *some* integration coverage, not just unit tests against fakes. Mocking the database often hides the bug.

## Don't test the framework

```go
// Bad: this tests json.Marshal, not your code
func TestUserToJSON(t *testing.T) {
    b, _ := json.Marshal(User{Name: "X"})
    if !strings.Contains(string(b), "X") {
        t.Error("...")
    }
}
```

Tests should exercise *your* logic. If a test only verifies that the standard library or a third-party library works as documented, it's a waste of CI time.

## Test fixtures and setup

`testMain` and `TestMain` are appropriate for setup that must happen exactly once per package (test DB migrations, mock servers). For per-test setup, use `t.Cleanup` to register teardown — it runs even if the test panics or fails. Avoid hand-rolled `defer` for setup that should follow the test lifecycle precisely.

## Goroutine leaks in tests

A test that spawns a goroutine and doesn't wait for it to complete leaks. Across a test suite, this accumulates and can mask real leaks in the SUT. Use `t.Cleanup` to signal cancellation and `sync.WaitGroup.Wait` to ensure all spawned goroutines have exited before the test returns.
