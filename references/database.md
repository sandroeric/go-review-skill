# Database

Database code is where correctness, performance, and data integrity converge. Go's `database/sql` is low-level; the patterns below are easy to get wrong and expensive to fix.

## Always check `rows.Err()` after iteration

`for rows.Next() { ... }` ends when either there are no more rows *or* an error occurred mid-iteration. `rows.Err()` distinguishes these.

**Bad:**
```go
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users")
if err != nil {
    return err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    rows.Scan(&u.ID, &u.Name)
    users = append(users, u)
}
return users, nil // ← silently returns partial results on iteration error
```

**Good:**
```go
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users")
if err != nil {
    return nil, err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name); err != nil {
        return nil, fmt.Errorf("scanning row: %w", err)
    }
    users = append(users, u)
}
if err := rows.Err(); err != nil {
    return nil, fmt.Errorf("iterating rows: %w", err)
}
return users, nil
```

Both `Scan` and `rows.Err()` errors matter — they signal different failures (decoding vs. transport).

## `defer tx.Rollback()` immediately after `BeginTx`

`Tx.Rollback()` is idempotent after a successful `Commit` — calling it returns `sql.ErrTxDone`, which you can ignore. This means you can always set up the rollback as a defer and let the success path commit explicitly.

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
    return err
}
defer tx.Rollback() // safe even after Commit

// ... operations against tx ...

if err := tx.Commit(); err != nil {
    return fmt.Errorf("committing: %w", err)
}
return nil
```

Without the defer, any early return that misses a commit leaks the transaction. The DB will eventually time it out, but you've held a row lock or written uncommitted data for that interval.

## Use ctx-aware variants

Always use the `Context` variants:

- `QueryContext` not `Query`
- `QueryRowContext` not `QueryRow`
- `ExecContext` not `Exec`
- `BeginTx(ctx, ...)` not `Begin()`
- `PingContext` not `Ping`

These honor `ctx.Done()` and cancel the underlying query when the context is cancelled. Without them, a slow query keeps running on the DB after the request has returned an error to the user, consuming a connection slot.

## N+1 query patterns

The classic shape: fetch a list, then for each item fetch related data in a loop.

**Bad:**
```go
users, _ := db.QueryUsers(ctx)
for _, u := range users {
    orders, _ := db.QueryOrdersByUser(ctx, u.ID) // ← N queries
    u.Orders = orders
}
```

**Good:**
```go
users, _ := db.QueryUsers(ctx)
ids := collectIDs(users)
ordersByUser, _ := db.QueryOrdersForUsers(ctx, ids) // ← 1 query
for i := range users {
    users[i].Orders = ordersByUser[users[i].ID]
}
```

The N+1 shape is particularly insidious because it works fine with 10 users in dev and falls over with 10,000 in production. Always flag it.

## Missing `LIMIT` on user-driven queries

Any query that can return an unbounded result set under user control is a memory exhaustion vector:

```go
// Bad: an attacker (or an unlucky filter combination) returns the whole table
db.QueryContext(ctx, "SELECT * FROM events WHERE user_id = $1", userID)

// Good: explicit pagination
db.QueryContext(ctx, "SELECT * FROM events WHERE user_id = $1 LIMIT $2 OFFSET $3", userID, limit, offset)
```

For internal/admin queries with predictable result-set sizes, `LIMIT` is less critical but still good hygiene.

## Prepared statement reuse

For queries executed many times with different parameters, `db.PrepareContext` returns a `*sql.Stmt` that can be reused without re-parsing. For one-shot queries, `QueryContext` / `ExecContext` are simpler — the connection pool internally caches prepared statements when possible.

Flag explicit `Prepare` / `Stmt` patterns only when they're used incorrectly (preparing inside a hot loop, not closing the statement, sharing a `Stmt` across DB instances).

## Driver-level cancellation

When a context is cancelled, the driver should signal cancellation to the database. Most modern drivers do (pgx, lib/pq with recent versions, go-sql-driver/mysql with `interpolateParams=false`). Reviewer asks: is the driver in use known to honor `ctx`? If not, a "cancelled" request is still consuming DB resources until the natural query timeout.

## Error mapping

Map driver-specific errors to domain errors at the repository boundary:

```go
func (r *UserRepo) Get(ctx context.Context, id string) (*User, error) {
    var u User
    err := r.db.QueryRowContext(ctx, "SELECT ... WHERE id = $1", id).Scan(...)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrUserNotFound // domain error
    }
    if err != nil {
        return nil, fmt.Errorf("querying user %s: %w", id, err)
    }
    return &u, nil
}
```

Callers should never need to `import "database/sql"` just to check `sql.ErrNoRows`. The repository layer translates.

## Connection pool tuning

`db.SetMaxOpenConns`, `db.SetMaxIdleConns`, `db.SetConnMaxLifetime`, `db.SetConnMaxIdleTime` should be set explicitly. Defaults are not production-tuned (unlimited connections, default idle time too long for managed databases that close idle connections aggressively).

Flag DB initialization that doesn't set these as MAJOR for service code, MINOR for tools/scripts.

## SQL injection (cross-reference)

Parameterized queries are mandatory for any value derived from user input. See `references/security.md` for the threat model; the engineering rule is simple: never use `fmt.Sprintf` to build a SQL query that includes user data.

## Transactions: keep them short

Long-running transactions hold locks and block other writers. Keep the work inside a transaction minimal: read, compute, write, commit. Don't make HTTP calls inside a transaction; don't loop over an unbounded set inside a transaction.
