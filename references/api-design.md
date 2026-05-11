# API design

API design findings are about long-term cost: an awkward API today is read, used, and worked around for years. Senior review focuses on the shape of the exported surface — types, interfaces, signatures, naming — because those are the parts that are most expensive to change after release.

## Accept interfaces, return concrete types

Functions should accept the smallest interface that captures what they need, and return the concrete type they construct. This gives callers maximum flexibility (they can pass anything that satisfies the interface) and gives the function maximum room to add fields later (the return type's method set can grow without breaking callers).

**Bad:**
```go
func NewStore(db *sql.DB) Storage { ... } // returns an interface — caller loses access to future methods
```

**Good:**
```go
type Reader interface{ Read(p []byte) (int, error) }

func Process(r Reader) *Result { ... } // accepts smallest interface, returns concrete
```

Exception: when the return is genuinely polymorphic at the API boundary (e.g., a factory that returns different concrete types based on config), an interface return is justified.

## Small interfaces

A Go interface with more than ~3 methods is a smell. The famous standard-library examples — `io.Reader`, `io.Writer`, `io.Closer`, `error`, `fmt.Stringer` — are single-method. The power comes from composition (`io.ReadCloser` is `Reader + Closer`).

A 10-method interface "Service" is rarely satisfied by anything except the one struct it was designed for. That means tests need a 10-method fake that no one reuses. Break it into smaller interfaces named for what they do, not what type implements them: `UserReader`, `UserWriter`, `UserDeleter`.

## No `Get` prefix on getters

Go convention: `user.Name()` not `user.GetName()`. The `Get` prefix is Java/JavaBean baggage. Use it only when the getter has side effects or returns multiple values where "Get" disambiguates from another method.

Setters get verbs: `SetName(s string)`. Constructors are `New` + the type: `NewUser`.

## Receiver consistency on a type

A type's method set should use a consistent receiver kind — either all value receivers or all pointer receivers. Mixing them creates subtle issues:

- An interface is satisfied by `T` only if all of `T`'s methods have value receivers. If even one is a pointer receiver, only `*T` satisfies the interface.
- A method with a value receiver receives a copy. Mutations don't survive. Hiding a value-receiver mutator among pointer-receiver methods is confusing.

Default to pointer receivers when the type has any mutating methods or holds a `sync.Mutex` (which must not be copied). Default to value receivers for small, immutable types where you want the calling convention to communicate "this is a value, not an identity."

## Useful zero values

A struct whose zero value is usable saves callers from constructor ceremony.

```go
// Good: var b bytes.Buffer; b.Write(...) works without initialization
type Buffer struct { ... }

// Bad: zero value panics or misbehaves; callers must remember to call NewX()
```

When designing a type, ask: "If a caller declares `var x MyType` and uses it, what happens?" If the answer is "panics" or "subtly wrong," consider whether the zero value can be made meaningful.

## Options pattern over many positional arguments

A function with 5+ positional arguments, especially when several are optional, is a readability hazard. Callers can't tell what each argument means without referring to the docs.

**Bad:**
```go
client := NewClient("https://api.example.com", 10*time.Second, 3, true, false, "v2")
```

**Good (functional options):**
```go
client := NewClient("https://api.example.com",
    WithTimeout(10*time.Second),
    WithRetries(3),
    WithGzip(true),
)
```

The options pattern is verbose to define but pays for itself with two or more call sites. For 2-3 args, a single config struct is often simpler:

```go
client := NewClient(Config{
    URL:     "...",
    Timeout: 10 * time.Second,
})
```

## Avoid `util`, `common`, `helpers` package names

These names don't describe what's inside. They accumulate unrelated code that should live elsewhere. Better names answer "what is this package for?": `validation`, `pathutil`, `httpcommon` (if there really is shared HTTP code), `mathutil` (when there's a real cluster of math helpers).

If you're tempted to write `package util`, ask whether the contents belong in the packages that use them, or whether there's a more specific name that captures their purpose.

## Minimize exported surface

Every exported symbol is a commitment: callers will use it, and changing it later breaks them. Default to lowercase. Export only when a caller outside the package needs the symbol.

Common over-exports:
- Helper functions used only inside the package.
- Constants used only inside the package.
- Types returned only inside the package.

When reviewing a new package, scan the exported names and ask, for each one, "does an external caller actually need this?"

## Package naming

Package names are lowercase, single-word, descriptive. The package name is a prefix to every reference (`user.Service`, `user.New`), so:

- Avoid stutter: `user.UserService` reads as `user.UserService`. Drop the prefix: `user.Service`.
- Avoid generic names: `helpers`, `utils`, `lib`, `common`.
- Avoid plurals when the package contains one concept: `model` not `models`.
- Match the file path: package at `internal/users/` is `users`, not `user`.

## Method naming consistency

Within a package, name similar operations consistently:

```go
store.Get(id)         // not GetUser
store.GetByEmail(em)  // ...unless disambiguating from Get(id)
store.Create(u)       // not Save / Insert / Put / Add — pick one
store.Delete(id)      // not Remove / Destroy
```

Inconsistency is a readability tax. Reviewer flags new methods that introduce a synonym for an existing operation.
