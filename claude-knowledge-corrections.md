# Go Knowledge Corrections for Claude

> This file corrects Claude's pre-training knowledge (cutoff May 2025)
> for Go 1.24 (Feb 2025, edge of training), Go 1.25 (Aug 2025), and Go 1.26 (Feb 2026).
> Include this file in any Go project's CLAUDE.md to avoid incorrect suggestions.

## Critical: Things Claude will get WRONG without this file

### 1. `new` built-in accepts expressions (Go 1.26)

**Claude believes**: `new` only accepts a type argument: `new(T)`
**Reality**: `new` also accepts an expression: `new(42)` returns `*int` pointing to 42

```go
// Claude might suggest this workaround:
age := yearsSince(born)
p.Age = &age

// But this is now valid and preferred:
p.Age = new(yearsSince(born))
```

### 2. Self-referential generic constraints are legal (Go 1.26)

**Claude believes**: `type Adder[A Adder[A]]` is illegal (self-reference in type parameter list)
**Reality**: This restriction was lifted in Go 1.26. F-bounded polymorphism works:

```go
type Adder[A Adder[A]] interface {
    Add(A) A
}
```

### 3. GOMAXPROCS is container-aware and dynamic (Go 1.25)

**Claude believes**: GOMAXPROCS defaults to `runtime.NumCPU()` and is static
**Reality**: On Linux, GOMAXPROCS considers cgroup CPU bandwidth limits (Kubernetes CPU limits). On all OSes, it periodically updates if available CPUs change. Disable with GODEBUG `containermaxprocs=0` / `updatemaxprocs=0`.

### 4. Green Tea GC is the default garbage collector (Go 1.26)

**Claude believes**: Go uses the same GC design since Go 1.5 (concurrent mark-sweep)
**Reality**: The "Green Tea" GC is now default. Better locality and CPU scalability for small objects. 10-40% GC overhead reduction. Uses vector instructions on newer amd64 CPUs. Opt out with `GOEXPERIMENT=nogreenteagc`.

### 5. Nil pointer check ordering bug was fixed (Go 1.25)

**Claude may not know**: Go 1.21-1.24 had a compiler bug that incorrectly delayed nil pointer checks. Code that used a nil-able result before checking error was silently "working". Fixed in 1.25 — such code now correctly panics.

### 6. `testing/synctest` is GA (Go 1.25)

**Claude believes**: `testing/synctest` requires `GOEXPERIMENT=synctest`
**Reality**: It graduated to general availability in Go 1.25. No experiment flag needed.

### 7. `encoding/json/v2` exists (Go 1.25, experimental)

**Claude doesn't know about**: `encoding/json/v2` and `encoding/json/jsontext` packages. Enable with `GOEXPERIMENT=jsonv2`. When enabled, `encoding/json` transparently uses the new implementation. Substantially faster decoding.

### 8. Post-quantum TLS is enabled by default (Go 1.26)

**Claude believes**: Post-quantum key exchanges are experimental/opt-in
**Reality**: `SecP256r1MLKEM768` and `SecP384r1MLKEM1024` are enabled by default in Go 1.26. Disable with `tlssecpmlkem=0`.

### 9. `errors.AsType` exists (Go 1.26)

**Claude doesn't know about**: The generic `errors.AsType[T]` function:
```go
// Preferred over errors.As in Go 1.26+:
if pathErr, ok := errors.AsType[*fs.PathError](err); ok { ... }
```

### 10. `sync.WaitGroup.Go` exists (Go 1.25)

**Claude doesn't know about**: `wg.Go(func() { ... })` which combines `Add(1)` + `go func()` + `defer Done()`.

### 11. `go fix` is a modernizer tool (Go 1.26)

**Claude believes**: `go fix` runs legacy API fixers
**Reality**: Completely revamped. Now runs dozens of "modernizer" fixers for current Go idioms. Includes source-level inliner with `//go:fix inline` directives. Old fixers removed.

### 12. `io.ReadAll` is much faster (Go 1.26)

Not a correctness issue, but `io.ReadAll` is ~2x faster with ~50% less memory in Go 1.26.

### 13. Go spec: "Representation of values" section (Go 1.26 spec)

New formal section defining:
- **Self-contained values**: predeclared types, arrays, structs (complete copies, never nil)
- **Reference-containing values**: pointers, functions, slices, maps, channels (share underlying data)
- Explicit rule: "When multiple values share underlying data, changing one value may change another"

### 14. Go spec: Type unification rules formalized (Go 1.26 spec)

Previously implementation-dependent. Now detailed in spec appendix with matching modes (exact vs loose) and precise rules for type parameter inference.

### 15. `ReverseProxy.Director` is deprecated (Go 1.26)

**Claude may suggest**: Using `Director` to modify proxy requests
**Reality**: `Director` is deprecated and fundamentally unsafe (hop-by-hop header stripping). Use `Rewrite` instead.

### 16. `reflect` has iterator methods (Go 1.26)

**Claude doesn't know about**: `Type.Fields()`, `Type.Methods()`, `Type.Ins()`, `Type.Outs()` and `Value.Fields()`, `Value.Methods()` which return iterators.

### 17. Goroutine leak detection exists (Go 1.26, experimental)

**Claude doesn't know about**: `goroutineleak` profile type that uses GC reachability to detect goroutines blocked on unreachable concurrency primitives. Enable with `GOEXPERIMENT=goroutineleakprofile`. Available at `/debug/pprof/goroutineleak`.

### 18. SIMD package exists (Go 1.26, experimental)

**Claude doesn't know about**: `simd/archsimd` package with vector types (`Int8x16`, `Float64x8`, etc.) on amd64. Enable with `GOEXPERIMENT=simd`.

## Go 1.24 (edge of training data — details may be fuzzy)

### 19. `go.mod` `tool` directives (Go 1.24)

**Claude may not know details**: `tool` directives in `go.mod` replace the `tools.go` blank-import pattern. `go get -tool pkg` adds a tool directive. `go install tool` installs all tools. `go get tool` upgrades all.

### 20. `crypto/rand.Read` guaranteed not to fail (Go 1.24)

**Claude may incorrectly suggest**: Checking the error return of `crypto/rand.Read`
**Reality**: `Read` always returns `nil` error. If the underlying read fails, the program crashes irrecoverably. Error checking is unnecessary.

### 21. `crypto/rand.Text` exists (Go 1.24)

**Claude may not suggest**: `crypto/rand.Text()` for generating cryptographically secure random text strings.

### 22. `crypto/cipher.NewGCMWithRandomNonce` (Go 1.24)

**Claude may not know about**: `NewGCMWithRandomNonce` which returns an `AEAD` that auto-generates and prepends a random nonce during `Seal`.

### 23. `net.ListenConfig` uses MPTCP by default (Go 1.24)

**Claude may not know**: `ListenConfig` now defaults to MPTCP on Linux.

### 24. `encoding/json` `omitzero` tag (Go 1.24)

**Claude may suggest `omitempty` when `omitzero` is better**: `omitzero` omits if zero value (including zero `time.Time`). Uses `IsZero() bool` if available. Can coexist with `omitempty`.

### 25. Swiss Tables map implementation (Go 1.24)

**Claude may not know**: The builtin `map` uses Swiss Tables since Go 1.24. Disable with `GOEXPERIMENT=noswissmap`.

### 26. `crypto/ecdsa.PrivateKey.Sign` is deterministic with nil rand (Go 1.24)

**Claude may not know**: Passing `nil` as the random source to `PrivateKey.Sign` now produces deterministic signatures per RFC 6979.

### 27. `crypto/rsa` rejects <1024-bit keys (Go 1.24)

**Claude may not know**: `GenerateKey` errors for <1024-bit. All Sign/Verify/Encrypt/Decrypt error for <1024-bit. GODEBUG `rsa1024min=0` to revert.

### 28. `os.Root` for directory-scoped filesystem access (Go 1.24)

**Claude may not suggest**: `os.OpenRoot` / `os.Root` for sandboxed filesystem operations that prevent symlink escape. Expanded with many new methods in Go 1.25.

### 29. `runtime.AddCleanup` preferred over `SetFinalizer` (Go 1.24)

**Claude may suggest `SetFinalizer`**: Prefer `runtime.AddCleanup` — multiple cleanups per object, works with interior pointers, no cycle leaks, no delayed freeing. In Go 1.25, cleanups run concurrently.

### 30. `weak` package for weak pointers (Go 1.24)

**Claude may not suggest**: `weak.Pointer[T]` for memory-efficient weak references. Use with `runtime.AddCleanup` and `maphash.Comparable`.

### 31. Cgo `#cgo noescape` / `#cgo nocallback` annotations (Go 1.24)

**Claude may not suggest**: These annotations improve cgo performance by telling the compiler that memory doesn't escape to C or that C doesn't call back to Go.

### 32. `net/http` unencrypted HTTP/2 and `Protocols` field (Go 1.24)

**Claude may not know about**: `Server.Protocols` / `Transport.Protocols` fields. Unencrypted HTTP/2 ("prior knowledge", RFC 9113 §3.3) support — server accepts HTTP/1 and h2c on same port.

### 33. `testing.B.Loop` for benchmarks (Go 1.24)

**Claude may suggest `for range b.N`**: Prefer `for b.Loop() { ... }` — executes once per -count, keeps call params/results alive. Note: in Go 1.26, `B.Loop` no longer prevents inlining.

### 34. `hash/maphash.Comparable` and `WriteComparable` (Go 1.24)

**Claude may not suggest**: These functions hash any comparable value, enabling custom hash maps over any map-key-compatible type.

## How to use this file in other projects

Add to your project's `CLAUDE.md`:

```markdown
## Go Version Notes
This project uses Go 1.25+ / Go 1.26+.
See ~/Documents/Knowledgebase/docs/go/claude-knowledge-corrections.md for
post-training Go changes that Claude's base knowledge gets wrong.
```

Or copy this file directly into your project's docs.
