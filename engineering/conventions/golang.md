---
type: Convention
title: Idiomatic Go
description: The Go conventions we write and review against — clear over clever, discovered interfaces, errors as values, communicating over sharing memory, table-driven tests, and current standard-library idioms over third-party reinventions.
tags: [convention, go]
timestamp: 2026-07-19T10:30:00Z
---

# The principles

* **Clear over clever.** Code should be obvious. If a function must be read three
  times to follow its control flow, rewrite it. Favor readability and simplicity;
  avoid abstraction that pays for itself later, if ever.
* **Errors are values.** They are not exceptions to catch — they are values to
  handle. Check them explicitly, at the point they occur.
* **Useful zero value.** Design types so the zero value works without
  initialization (`sync.Mutex`, `bytes.Buffer`). It removes constructors and
  init boilerplate.
* **Early return.** Handle errors and edge cases immediately and return. Keep the
  happy path unindented; do not put the main logic in an `else`.

# Package organization

* **Start flat.** A single package alongside `main.go` is the default. Add a
  package only when you need a new namespace for clarity or to decouple a genuinely
  independent domain.
* **Name packages for what they do, not their layer.** `jobs/`, `auth/`,
  `billing/` — never `service/`, `repository/`, `controller/`, `domain/`. The
  layer-named "Clean Architecture / DDD" split causes circular imports and
  interface proliferation and adds no clarity.
* **Reject `utils/`, `helpers/`, `common/`.** They signal unclear ownership.
* **Stay one level deep.** Each package owns one concern, describable in a sentence
  without reference to the others. `main` is the only wiring point — no sideways
  imports between domain packages. Sub-concerns that always travel together stay in
  one package (job creation and its worker handler; HTTP handlers and their
  templates).
* **`internal/` is for libraries.** It is a compiler-enforced import boundary. In a
  library, use it to share types between your own packages while denying end users.
  In an application nobody can import anyway, it only adds path depth.

# Interfaces

* **Discover interfaces; do not design them upfront.** Write concrete types first.
  Introduce an interface only when a consumer genuinely needs interchangeable
  implementations.
* **Define the interface where it is used**, in the consuming package — not
  alongside the implementation. The concrete type need not know the interface
  exists.
* **Accept interfaces, return structs.** Take the smallest interface you need
  (`io.Reader`, not `*os.File`); return concrete types so callers are not forced
  into type assertions.

# Library API design

When a library wraps a stateful resource (a vault, database, config store), make
that resource struct the entry point, and have its methods return domain-typed
sub-objects:

```go
v, err := vault.Open(path)
idx, err := v.People()           // *people.Index
note, err := v.Daily(time.Now()) // *daily.Note
```

* One import to start; sub-packages hold the rich domain types.
* **No package-level global state** in library code — it breaks concurrency safety
  and testability. (A CLI-only tool with genuinely one instance, like a flag set,
  is the sole exception.)
* Don't pass the resource/config struct as the first argument to every function.

# Error handling

Wrap errors with context about what was attempted, using `%w` to preserve the
chain — not to synthesize stack traces:

```go
if err != nil {
    return fmt.Errorf("loading config file %s: %w", path, err)
}
```

Join independent errors with `errors.Join` (not `fmt.Errorf("%w; %w", …)` or a
`multierr` package); test with `errors.Is` / `errors.As`.

# Concurrency

* **Share memory by communicating.** Prefer passing data over a channel to guarding
  it with a mutex. Channels orchestrate; mutexes serialize.
* **Never start a goroutine without knowing how it stops.** Every `go func()` needs
  a clear exit, usually a `context.Context` or a closed channel.
* **Bound concurrency with `errgroup` + `SetLimit`** rather than hand-rolled
  semaphore channels or static worker pools. When no error propagation is needed,
  `sync.WaitGroup.Go` removes the Add/Done boilerplate.
* Loop variables are per-iteration since Go 1.22 — never write the old `x := x`
  capture.
* Use typed atomics (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) over the
  function-based `sync/atomic` API.
* `context.WithoutCancel` detaches cancellation while keeping values, for background
  work that must outlive a request.

# Configuration

For a struct with many optional parameters, use functional options rather than a
sprawling constructor:

```go
func NewServer(addr string, opts ...Option) *Server { … }
srv := NewServer(":8080", WithTimeout(5*time.Second))
```

# Testing

Testing in Go is Go programming — no BDD frameworks (Ginkgo), no heavy mocking.

* **Table-driven tests** with `t.Run(tt.name, …)` are the standard.
* Call **`t.Helper()`** in assertion helpers so failures point at the test case.
* Prefer **hand-written fakes/stubs** over generated mocks; Go's implicit
  interfaces make them cheap.
* Use **`cmp`** (`github.com/google/go-cmp/cmp`) for struct/map comparison and
  readable diffs — not `reflect.DeepEqual`.
* Put large or complex fixtures in a **`testdata/`** directory (the tool ignores
  it); compare against golden files.
* **Abstract the filesystem** (e.g. `afero.Fs`) instead of hardcoding `os` deep in
  logic; inject an in-memory FS in tests.
* Never `time.Sleep` to wait for a goroutine. Use channels, explicit
  synchronization, or `testing/synctest` (fake clock, deterministic timing).
* Modern helpers: `t.Context()` (canceled at test end), `t.Chdir(dir)` (restored
  after), and `for b.Loop()` for benchmarks (replaces `b.N`, defeats dead-code
  elimination).

# Generics

Generics exist to remove duplicated **algorithms**, not to build type hierarchies.
Thinking in inheritance or polymorphism means writing Java in Go.

* **Use** them when the same algorithm runs over multiple concrete types
  (`Map[S, T]`, `Min[T cmp.Ordered]`). Use `comparable` for keys/equality and
  `cmp.Ordered` for ordering.
* **Don't** create generic base types, services, or repositories, and don't use
  `any` as a constraint to mean "type decided later" — that's a design smell.
* Start concrete; generify only when the same logic repeats across 3+ types.

# Prefer the current standard library

Reach for stdlib before a third-party utility or a hand-rolled helper.

| Instead of | Use |
| --- | --- |
| `sort.Slice` | `slices.Sort`, `slices.SortFunc` |
| manual contains/index loops | `slices.Contains`, `slices.Index`, `slices.ContainsFunc` |
| manual key/value loops | `maps.Keys`, `maps.Values`, `maps.Clone`, `maps.Equal`; `slices.Sorted(maps.Keys(m))` |
| `if x == 0 { x = default }` | `cmp.Or(x, default)`; also `cmp.Compare`, built-in `min`/`max` |
| `multierr` / `fmt.Errorf("%w; %w")` | `errors.Join` |
| custom `Next()/HasNext()` iterators | return `iter.Seq[T]` / `iter.Seq2[K,V]`, ranged with plain `range` |
| `math/rand` | `math/rand/v2` (`rand.IntN`, `rand.N`) |
| `omitempty` for zero times/structs | `omitzero` (Go 1.24) |

Prefer returning an iterator over allocating a slice for large or lazily-produced
sequences.

# HTTP services

* **Use the stdlib router.** Since Go 1.22 `net/http.ServeMux` does method and
  path-parameter routing (`"GET /users/{id}"`, `r.PathValue("id")`,
  `"/files/{path...}"`). Reach for chi/gorilla only for named-route generation or
  regex constraints.
* **Always set timeouts.** `http.ListenAndServe` has none — one slow client holds a
  connection forever (slow-loris). Set `ReadHeaderTimeout`, `ReadTimeout`,
  `WriteTimeout`, `IdleTimeout` on an `http.Server`. Outbound `http.DefaultClient`
  has no timeout either.
* **Provide graceful shutdown.** On a signal (`signal.NotifyContext`), stop
  accepting and drain in-flight requests with `srv.Shutdown` using a *fresh*
  deadline context. Long-lived connections (SSE, websockets) must watch
  `r.Context()`.
* **Middleware is `func(http.Handler) http.Handler`.** Compose functions; don't
  import a middleware framework.

# Logging

Use `log/slog` for structured logs.

* Pass `*slog.Logger` as a dependency; **no package-level logger global beyond
  `main`** (`slog.Default()` is a fallback only there).
* **Never log *and* return an error** — log at the boundary, return the error up the
  stack.
* Attach context with `logger.With(...)` and `slog.Group(...)`. Levels: `Debug`
  (internal, high volume), `Info` (lifecycle), `Warn` (recoverable), `Error` (needs
  attention).

# Current syntax

* `for i := range 10` for counted loops (Go 1.22).
* `//go:build linux || darwin` — never the deprecated `// +build`.
* `any`, never `interface{}`.
* Track dev tools with the `tool` directive (`go get -tool …`, `go tool …`) — never
  a `tools.go` with blank imports.

# Trust the toolchain

The Go tool is extremely reliable and almost never the source of a bug. `go run`
always recompiles, `go build` is deterministic, and the build cache is keyed by
source content. When something looks wrong, the cause is almost always in the code:
the edit didn't fix the logic, was made in the wrong file/package, missed a second
call site, or the error comes from a different path. Re-read the (accurate) error
message and confirm the compiled file with `go list -f '{{.GoFiles}}' .` before
suspecting the tool. Do not clear the build cache or restart the toolchain as a
first move.

# Anti-patterns to reject

- [ ] Deep directory trees or `internal/` sprawl for "Clean Architecture"
- [ ] Layer-named packages (`service/`, `repository/`, `controller/`, `domain/`)
- [ ] `utils/`, `helpers/`, `common/` packages
- [ ] Static worker pools instead of `errgroup` with `SetLimit`
- [ ] Goroutines with no defined stop condition
- [ ] Generic interfaces for polymorphism instead of concrete types
- [ ] `any` as a constraint meaning "type decided later"
- [ ] `sort.Slice` where `slices.Sort` fits; `reflect.DeepEqual` where `cmp` fits
- [ ] Old `// +build` constraints; `interface{}` over `any`; `math/rand` over `v2`
- [ ] `tools.go` blank-import files instead of the `tool` directive
- [ ] Custom `Next()/HasNext()` iterators; allocating a slice where an iterator fits
- [ ] Package-level logger globals outside `main`; logging *and* returning an error
- [ ] BDD/heavy mocking frameworks; `time.Sleep` in tests to await goroutines
- [ ] HTTP servers without timeouts or without a graceful-shutdown path
- [ ] Middleware frameworks where function composition suffices
- [ ] Suspecting the Go toolchain before exhausting code-level explanations

# Source

Distilled from spf13's Go skill:
`https://github.com/spf13/go-skills/blob/main/go/SKILL.md`.
