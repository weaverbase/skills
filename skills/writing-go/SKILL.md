---
name: writing-go
description: Use when writing, changing, or reviewing Go code that handles or wraps errors, uses panic, starts goroutines or fans out work, passes context.Context, builds HTTP servers or routes, logs, defines interfaces or generics, writes tests or benchmarks, encodes JSON, or manages go.mod tools; also when tempted to use interface{}, ioutil, log.Printf, gorilla/mux, sort.Slice, or other pre-generics Go idioms.
---

# Writing Go

## Overview

Write current, idiomatic Go: explicit errors with context, `context.Context` through every I/O call, goroutines that always exit, the standard library first, and language features that match the module's `go` version.

**Core principle:** Use the newest idiom the module's `go` directive allows. Old tutorials and older training data predate generics, `slog`, `slices`, and the method-aware `ServeMux`; familiarity is not a reason to keep their patterns.

These rules apply to new or requested changes. Do not migrate existing working code unless asked. Follow the project's existing choices (router, logger, test helpers, linters) over these defaults.

## Check the Go Version First

Read the `go` line in `go.mod` before using a feature. Use only what that version supports; the "Since" column below is the minimum. Do not raise the `go` line just to use a newer feature unless the change asks for it.

## Quick Reference

| Area | Do | Do not | Since |
| --- | --- | --- | --- |
| Empty interface, when one is needed | `any` | `interface{}`; `any` where a concrete type or type parameter fits | 1.18 |
| File and I/O helpers | `os.ReadFile`, `os.WriteFile`, `io.ReadAll` | `io/ioutil` | 1.16 |
| Slices and maps | `slices.Sort`, `slices.Contains`, `slices.Index`, `maps.Keys`, `min`, `max` | `sort.Slice` for simple sorts, hand-written contains/min/max loops | 1.21 |
| Logging | `log/slog` with structured attributes | `log.Printf`, `fmt.Println` for application logs | 1.21 |
| HTTP routing | `http.NewServeMux` with `"GET /items/{id}"` and `r.PathValue("id")` | A third-party router only for method and path matching | 1.22 |
| Counting loops | `for i := range n` | `for i := 0; i < n; i++` when only counting | 1.22 |
| Loop variables | Use the loop variable directly in closures | `v := v` copies | 1.22 |
| Test context | `t.Context()` | `context.Background()` with manual cancel in tests | 1.24 |
| Benchmarks | `for b.Loop() { ... }` | `for i := 0; i < b.N; i++` | 1.24 |
| JSON zero values | `json:",omitzero"` for `time.Time` and structs | `omitempty`, which never omits them | 1.24 |
| Dev tools | `tool` directive in `go.mod`, run with `go tool <name>` | A blank-import `tools.go` file | 1.24 |
| Untrusted file paths | `os.OpenRoot(dir)` and the `Root` methods | Hand-rolled `..` checks | 1.24 |
| WaitGroup | `wg.Go(func() { ... })` | `wg.Add(1)` plus `defer wg.Done()` | 1.25 |
| GOMAXPROCS in containers | The default (container-aware) | Adding `automaxprocs` | 1.25 |
| Typed error matching | `errors.AsType[*MyErr](err)` | `var e *MyErr; errors.As(err, &e)` | 1.26 |
| Modernizing code | `go fix ./...` | Manual rewrites to newer idioms | 1.26 |

## Errors

- Return errors; do not `panic` for recoverable failures. `Must*` helpers are for package-level initialization with constant input.
- Wrap with context using `%w`: `fmt.Errorf("load user %d: %w", id, err)`. Messages are lowercase, without trailing punctuation, and describe the operation that failed.
- Match with `errors.Is` for sentinels and `errors.As` (or `errors.AsType` on 1.26+) for error types. Never compare `err.Error()` strings.
- Handle an error once: either log it or return it, not both.
- Expected failures that callers branch on are exported sentinels (`var ErrNotFound = errors.New("not found")`) or error types. Keep them in the package that owns the behavior, and keep transport details (HTTP status, exit code) out of them.

```go
var ErrNotFound = errors.New("not found")

func (s *Users) Get(ctx context.Context, id int64) (User, error) {
	u, err := s.store.Get(ctx, id)
	if errors.Is(err, pgx.ErrNoRows) {
		return User{}, fmt.Errorf("user %d: %w", id, ErrNotFound)
	}
	if err != nil {
		return User{}, fmt.Errorf("get user %d: %w", id, err)
	}
	return u, nil
}
```

## Context and Concurrency

- `ctx context.Context` is the first parameter of any function that does I/O or may block. Pass it through; do not store it in a struct. Use `context.Background()` only in `main`, tests before 1.24, and top-level jobs.
- Every goroutine has a clear exit: it finishes, or it stops when its context is cancelled. Never start a goroutine you cannot stop.
- For fan-out where any error should cancel the rest, use `golang.org/x/sync/errgroup` with `errgroup.WithContext`, and `g.SetLimit(n)` to bound concurrency. For fan-out without errors, use `sync.WaitGroup` (`wg.Go` on 1.25+).
- Shared mutable state is protected by a mutex or owned by one goroutine. Run tests with `-race` when the code is concurrent.

```go
func fetchAll(ctx context.Context, urls []string) ([][]byte, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8)
	bodies := make([][]byte, len(urls))
	for i, url := range urls {
		g.Go(func() error {
			body, err := fetch(ctx, url)
			if err != nil {
				return fmt.Errorf("fetch %s: %w", url, err)
			}
			bodies[i] = body
			return nil
		})
	}
	return bodies, g.Wait()
}
```

## HTTP Servers

- Start with `net/http`. Its `ServeMux` matches methods and path wildcards since 1.22. Keep an existing chi, echo, or gin setup; do not add one only for routing.
- Set server timeouts. The zero-value `http.Server` has none, so slow clients can hold connections open.
- Shut down gracefully on SIGINT and SIGTERM with `signal.NotifyContext` and `srv.Shutdown`.

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	mux := http.NewServeMux()
	mux.HandleFunc("GET /users/{id}", getUser)

	srv := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	go func() {
		<-ctx.Done()
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		_ = srv.Shutdown(shutdownCtx)
	}()

	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		slog.Error("server stopped", "err", err)
		os.Exit(1)
	}
}
```

## Logging

Use `log/slog`. Create the logger once in `main` (`slog.NewJSONHandler` for production, `slog.NewTextHandler` for local runs), set it with `slog.SetDefault` or pass it in, and log with key-value attributes: `slog.Info("user created", "user_id", id)`. Keep an existing `zap` or `zerolog` setup. Never log secrets, tokens, or personal data.

## Interfaces and Generics

- Define small interfaces where they are consumed, not next to the implementation. Accept interfaces, return concrete types.
- Do not create an interface for a struct with one implementation and no test or boundary need for it.
- Use generics to remove real duplication across types. Do not make a function generic when it only ever handles one type.
- Avoid `any` for values whose type is known. It moves type checks to runtime assertions that can panic. Use a concrete type, a small interface, or a type parameter instead. `any` is fine as a generic constraint (`[T any]`), for `fmt`/`slog` arguments, and for JSON whose shape is truly unknown; decode known shapes into structs.

## Project Layout

Follow the official module layout guide: a single package at the root for small modules, `cmd/<name>/main.go` for each binary, and `internal/` for packages other modules must not import. Do not add `pkg/`, `src/`, or a layered directory tree by default; the popular "golang-standards/project-layout" repository is not an official standard.

## Testing and Tooling

- Use the standard `testing` package with table-driven tests and `t.Run` subtests. Keep `testify` or another helper library when the project already uses it.
- Format with `gofmt` (or `goimports`), and run `go vet ./...` and `go test ./...`. Run `golangci-lint` or `staticcheck` when the project configures them.
- Pin developer tools with the `tool` directive (`go get -tool <module>`) on 1.24+.

```go
func TestParsePort(t *testing.T) {
	tests := []struct {
		name    string
		in      string
		want    int
		wantErr bool
	}{
		{name: "valid", in: "8080", want: 8080},
		{name: "not a number", in: "http", wantErr: true},
		{name: "out of range", in: "70000", wantErr: true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := ParsePort(tt.in)
			if (err != nil) != tt.wantErr {
				t.Fatalf("ParsePort(%q) error = %v, wantErr %v", tt.in, err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("ParsePort(%q) = %d, want %d", tt.in, got, tt.want)
			}
		})
	}
}
```

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "`interface{}` and `ioutil` still compile." | Use `any` and `os`/`io`. Working code is not current convention. |
| "Add gorilla/mux or gin for path parameters." | `ServeMux` handles methods and wildcards on 1.22+. Keep an existing router; do not add one for this. |
| "`log.Printf` is enough." | Use `slog` with attributes, or the project's existing structured logger. |
| "Log the error here and return it too." | Handle it once: wrap and return, or log at the top. |
| "Check `strings.Contains(err.Error(), \"not found\")`." | Use `errors.Is` or `errors.As` with a sentinel or error type. |
| "Fire off a goroutine; it will finish eventually." | Give it a context or another exit path, and wait for it. |
| "Add an interface for every struct so it can be mocked." | Define a small interface at the consumer only when a test or boundary needs it. |
| "Use the newest feature; the toolchain supports it." | The module's `go` line decides, not the installed toolchain. |
| "Create `pkg/`, `internal/domain/`, `internal/infrastructure/` up front." | Start flat; add `cmd/` and `internal/` when there is a reason. |

## Red Flags

- `panic` used for an ordinary error path.
- A function parameter, return, or struct field typed `any` or `map[string]any` when the shape is known.
- An error returned without context, or wrapped with `%v` where callers need to match it.
- `context.Background()` or `context.TODO()` deep inside a call chain that received a context.
- A goroutine with no cancellation, no `WaitGroup` or `errgroup`, and no channel that ends it.
- `http.ListenAndServe(addr, mux)` in production code, with no timeouts.
- `fmt.Println` or `log.Printf` used for application logging in a new service.
