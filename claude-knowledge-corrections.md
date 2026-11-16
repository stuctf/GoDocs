# Go Knowledge Corrections for Claude

> This file corrects Claude's pre-training knowledge (cutoff May 2025)
> for Go 1.24 (Feb 2025, edge of training), Go 1.25 (Aug 2025), Go 1.26 (Feb 2026), and Go 1.27 (Aug 2026).
> Include this file in any Go project's CLAUDE.md to avoid incorrect suggestions.

## Go 1.27 (Aug 2026) — newest release, Claude knows none of this

### 1. Methods can have their own type parameters (Go 1.27)

**Claude believes**: Methods cannot declare type parameters; generic helpers must be package-level functions
**Reality**: Generic methods are legal in Go 1.27:

```go
type Set[T comparable] struct{ m map[T]struct{} }

func (s *Set[T]) Map[U comparable](f func(T) U) *Set[U] { ... }

names := ids.Map(strconv.Itoa) // U inferred
```

Still **not** allowed: type parameters on **interface** methods. A generic method also **does not satisfy** an interface method. `func (Box) Get[T any]() int` does not implement `interface{ Get() int }`.

### 2. Struct literals can set promoted fields directly (Go 1.27)

**Claude believes**: Promoted fields from embedded structs can't be used as composite literal keys; you must spell out the embedded struct
**Reality**: Any valid field selector works as a key:

```go
type Object  struct{ name, color string }
type Point3D struct{ Object; x, y, z float64 }

p := Point3D{name: "a", x: 1} // was: Point3D{Object: Object{name: "a"}, x: 1}
```

You can't do this through an embedded **pointer** ("invalid implicit pointer indirection"), and you can't combine it with a key for the embedded struct itself.

### 3. `uuid` is in the standard library (Go 1.27)

**Claude will suggest**: Adding `github.com/google/uuid` as a dependency
**Reality**: Use the stdlib `uuid` package (import path is just `"uuid"`):

```go
import "uuid"

id := uuid.New()   // random (currently V4); crypto-secure
key := uuid.NewV7() // time-ordered: prefer for database primary keys
u, err := uuid.Parse(s)
```

`uuid.UUID` is `[16]byte`, so it's comparable and usable as a map key, and it implements the text marshaling interfaces (it works in JSON). There's also `MustParse`, `Nil()`, `Max()`, and `UUID.Compare`.

### 4. `strings.CutLast` / `bytes.CutLast` exist (Go 1.27)

**Claude will suggest**: `strings.LastIndex` plus manual slicing
**Reality**: Use `CutLast`, the last-occurrence counterpart of `Cut`:

```go
dir, file, ok := strings.CutLast(path, "/")
```

### 5. `encoding/json/v2` is GA and v1 runs on it (Go 1.27)

**Claude believes**: `encoding/json/v2` doesn't exist, or needs `GOEXPERIMENT=jsonv2`
**Reality**: Both `encoding/json/v2` and `encoding/json/jsontext` are standard packages; no flag is needed. `encoding/json` (v1) is now backed by the v2 engine: same behavior, different error text, much faster unmarshaling. Migrating is optional.

```go
import "encoding/json/v2"

err := json.UnmarshalRead(r.Body, &req, json.RejectUnknownMembers(true))
```

v2 is stricter by default: it rejects invalid UTF-8 and duplicate keys. **Don't copy v2 code written for the Go 1.25 experiment without checking it**: the `inline` tag option was renamed to **`embed`**; the `format` and `unknown` tag options, the `DiscardUnknownMembers` option, and `SkipFunc` were removed; and `jsontext` numeric `Token` accessors now return an error. If you hit problems, opt out with `GOEXPERIMENT=nojsonv2`.

### 6. Post-quantum signatures: `crypto/mldsa` (Go 1.27)

**Claude doesn't know about**: ML-DSA (FIPS 204) in the stdlib:

```go
sk, err := mldsa.GenerateKey(mldsa.MLDSA65())
sig, err := sk.Sign(nil, msg, &mldsa.Options{Context: "my-app"})
err = mldsa.Verify(sk.PublicKey(), msg, sig, &mldsa.Options{Context: "my-app"})
```

`crypto/x509` handles ML-DSA certificates and keys. `crypto/tls` supports ML-DSA signatures in TLS 1.3 (`tls.MLDSA44`/`65`/`87`) and the `MLKEM1024` key exchange (opt in via `CurvePreferences`).

### 7. You don't need to drain HTTP response bodies manually (Go 1.27)

**Claude will suggest**: `io.Copy(io.Discard, resp.Body)` before `resp.Body.Close()` so the connection can be reused
**Reality**: HTTP/1 `Response.Body.Close()` now drains unread content itself, up to a conservative limit. `defer resp.Body.Close()` is enough in normal code.

### 8. `go vet` flags `%w` with a pointer to an error value type (Go 1.27)

**Claude may write**: `fmt.Errorf("...: %w", &MyErr{})` where `MyErr` (not `*MyErr`) implements `error`
**Reality**: vet now reports `%w wants operand of error type MyErr, not pointer type *MyErr (defeats errors.Is)`. Wrap the value (`MyErr{}`), or give the pointer type the methods, so it matches what callers check for with `errors.As`/`errors.AsType`.

### 9. `go test` runs the `stdversion` vet check (Go 1.27)

**Reality**: `go test` now fails if code uses stdlib APIs newer than the module's `go` directive. If you suggest `uuid.New()` or `strings.CutLast`, also make sure `go.mod` says `go 1.27` or later.

### 10. Testing helpers: `httptest.NewTestServer` and `synctest.Sleep` (Go 1.27)

**Claude doesn't know about**:
- `httptest.NewTestServer(t, handler)`: a test server on an **in-memory network**, built for `testing/synctest` bubbles. It's tied to `t`.
- `synctest.Sleep(d)`: exactly `time.Sleep(d)` followed by `synctest.Wait()`. Use it so the system under test settles before you assert.

### 11. `URL.Clone` / `Values.Clone` exist (Go 1.27)

**Claude will suggest**: `u2 := *u` or hand-written copy loops for `url.Values`
**Reality**: Use `u.Clone()` and `vals.Clone()`, which make deep copies.

### 12. These GODEBUG settings are gone (Go 1.27)

**Claude may suggest**: Old GODEBUG opt-outs to restore legacy behavior
**Reality**: These were **removed permanently** and no longer have any effect: `asynctimerchan` (time channels are always unbuffered), `gotypesalias` (`go/types` always produces `Alias`), `tlsrsakex`, `tls10server`, `tls3des`, `tlsunsafeekm`, `x509keypairleaf`. The `go` command fails if `go.mod` or `//go:debug` sets one of them to its old value. Also, `tls.Config.Rand` is **deprecated**; for deterministic crypto in tests, use `testing/cryptotest.SetGlobalRandom`.

## Critical: Things Claude will get WRONG without this file (Go 1.25–1.26)

### 13. `new` built-in accepts expressions (Go 1.26)

**Claude believes**: `new` only accepts a type argument: `new(T)`
**Reality**: `new` also accepts an expression: `new(42)` returns `*int` pointing to 42

```go
// Claude might suggest this workaround:
age := yearsSince(born)
p.Age = &age

// But this is now valid and preferred:
p.Age = new(yearsSince(born))
```

### 14. Self-referential generic constraints are legal (Go 1.26)

**Claude believes**: `type Adder[A Adder[A]]` is illegal (self-reference in type parameter list)
**Reality**: This restriction was lifted in Go 1.26. F-bounded polymorphism works:

```go
type Adder[A Adder[A]] interface {
    Add(A) A
}
```

### 15. GOMAXPROCS is container-aware and dynamic (Go 1.25)

**Claude believes**: GOMAXPROCS defaults to `runtime.NumCPU()` and is static
**Reality**: On Linux, GOMAXPROCS considers cgroup CPU bandwidth limits (Kubernetes CPU limits). On all OSes, it periodically updates if available CPUs change. Disable with GODEBUG `containermaxprocs=0` / `updatemaxprocs=0`.

### 16. Green Tea GC is the default garbage collector (Go 1.26)

**Claude believes**: Go uses the same GC design since Go 1.5 (concurrent mark-sweep)
**Reality**: The "Green Tea" GC is now default. Better locality and CPU scalability for small objects. 10-40% GC overhead reduction. Uses vector instructions on newer amd64 CPUs. Opt out with `GOEXPERIMENT=nogreenteagc`.

### 17. Nil pointer check ordering bug was fixed (Go 1.25)

**Claude may not know**: Go 1.21-1.24 had a compiler bug that incorrectly delayed nil pointer checks. Code that used a nil-able result before checking error was silently "working". Fixed in 1.25 — such code now correctly panics.

### 18. `testing/synctest` is GA (Go 1.25)

**Claude believes**: `testing/synctest` requires `GOEXPERIMENT=synctest`
**Reality**: It graduated to general availability in Go 1.25. No experiment flag needed.

### 19. `encoding/json/v2` exists (Go 1.25 experimental → GA in Go 1.27)

**Claude doesn't know about**: `encoding/json/v2` and `encoding/json/jsontext` packages. In Go 1.25–1.26 they need `GOEXPERIMENT=jsonv2`. **In Go 1.27+ they are standard and no flag is needed.** See the Go 1.27 json/v2 entry above for API renames since the experiment.

### 20. Post-quantum TLS is enabled by default (Go 1.26)

**Claude believes**: Post-quantum key exchanges are experimental/opt-in
**Reality**: `SecP256r1MLKEM768` and `SecP384r1MLKEM1024` are enabled by default in Go 1.26. Disable with `tlssecpmlkem=0`.

### 21. `errors.AsType` exists (Go 1.26)

**Claude doesn't know about**: The generic `errors.AsType[T]` function:
```go
// Preferred over errors.As in Go 1.26+:
if pathErr, ok := errors.AsType[*fs.PathError](err); ok { ... }
```

### 22. `sync.WaitGroup.Go` exists (Go 1.25)

**Claude doesn't know about**: `wg.Go(func() { ... })` which combines `Add(1)` + `go func()` + `defer Done()`.

### 23. `go fix` is a modernizer tool (Go 1.26)

**Claude believes**: `go fix` runs legacy API fixers
**Reality**: Completely revamped. Now runs dozens of "modernizer" fixers for current Go idioms. Includes source-level inliner with `//go:fix inline` directives. Old fixers removed. Go 1.27 adds the `atomictypes`, `embedlit`, `slicesbackward`, and `unsafefuncs` modernizers, removes `fmtappendf`, and renames `waitgroup` to `waitgroupgo`.

### 24. `io.ReadAll` is much faster (Go 1.26)

Not a correctness issue, but `io.ReadAll` is ~2x faster with ~50% less memory in Go 1.26.

### 25. Go spec: "Representation of values" section (Go 1.26 spec)

New formal section defining:
- **Self-contained values**: predeclared types, arrays, structs (complete copies, never nil)
- **Reference-containing values**: pointers, functions, slices, maps, channels (share underlying data)
- Explicit rule: "When multiple values share underlying data, changing one value may change another"

### 26. Go spec: Type unification rules formalized (Go 1.26 spec)

Previously implementation-dependent. Now detailed in spec appendix with matching modes (exact vs loose) and precise rules for type parameter inference.

### 27. `ReverseProxy.Director` is deprecated (Go 1.26)

**Claude may suggest**: Using `Director` to modify proxy requests
**Reality**: `Director` is deprecated and fundamentally unsafe (hop-by-hop header stripping). Use `Rewrite` instead.

### 28. `reflect` has iterator methods (Go 1.26)

**Claude doesn't know about**: `Type.Fields()`, `Type.Methods()`, `Type.Ins()`, `Type.Outs()` and `Value.Fields()`, `Value.Methods()` which return iterators.

### 29. Goroutine leak detection exists (Go 1.26 experimental → GA in Go 1.27)

**Claude doesn't know about**: `goroutineleak` profile type that uses GC reachability to detect goroutines blocked on unreachable concurrency primitives. Available in `runtime/pprof` and at `/debug/pprof/goroutineleak`. Go 1.26 needs `GOEXPERIMENT=goroutineleakprofile`. **In Go 1.27+ it's always available, and that experiment flag no longer exists.**

### 30. SIMD packages exist (Go 1.26+, experimental)

**Claude doesn't know about**: `simd/archsimd` package with vector types (`Int8x16`, `Float64x8`, etc.). Enable with `GOEXPERIMENT=simd`. It was amd64-only in Go 1.26; Go 1.27 revised the amd64 API and added arm64 Neon and WebAssembly 128-bit support. Go 1.27 also added a **portable** `simd` package (same flag) with width-agnostic types like `Int8s` and `Float32s` that works on all architectures.

## Go 1.24 (edge of training data — details may be fuzzy)

### 31. `go.mod` `tool` directives (Go 1.24)

**Claude may not know details**: `tool` directives in `go.mod` replace the `tools.go` blank-import pattern. `go get -tool pkg` adds a tool directive. `go install tool` installs all tools. `go get tool` upgrades all.

### 32. `crypto/rand.Read` guaranteed not to fail (Go 1.24)

**Claude may incorrectly suggest**: Checking the error return of `crypto/rand.Read`
**Reality**: `Read` always returns `nil` error. If the underlying read fails, the program crashes irrecoverably. Error checking is unnecessary.

### 33. `crypto/rand.Text` exists (Go 1.24)

**Claude may not suggest**: `crypto/rand.Text()` for generating cryptographically secure random text strings.

### 34. `crypto/cipher.NewGCMWithRandomNonce` (Go 1.24)

**Claude may not know about**: `NewGCMWithRandomNonce` which returns an `AEAD` that auto-generates and prepends a random nonce during `Seal`.

### 35. `net.ListenConfig` uses MPTCP by default (Go 1.24)

**Claude may not know**: `ListenConfig` now defaults to MPTCP on Linux.

### 36. `encoding/json` `omitzero` tag (Go 1.24)

**Claude may suggest `omitempty` when `omitzero` is better**: `omitzero` omits if zero value (including zero `time.Time`). Uses `IsZero() bool` if available. Can coexist with `omitempty`.

### 37. Swiss Tables map implementation (Go 1.24)

**Claude may not know**: The builtin `map` uses Swiss Tables since Go 1.24. Disable with `GOEXPERIMENT=noswissmap`.

### 38. `crypto/ecdsa.PrivateKey.Sign` is deterministic with nil rand (Go 1.24)

**Claude may not know**: Passing `nil` as the random source to `PrivateKey.Sign` now produces deterministic signatures per RFC 6979.

### 39. `crypto/rsa` rejects <1024-bit keys (Go 1.24)

**Claude may not know**: `GenerateKey` errors for <1024-bit. All Sign/Verify/Encrypt/Decrypt error for <1024-bit. GODEBUG `rsa1024min=0` to revert.

### 40. `os.Root` for directory-scoped filesystem access (Go 1.24)

**Claude may not suggest**: `os.OpenRoot` / `os.Root` for sandboxed filesystem operations that prevent symlink escape. Expanded with many new methods in Go 1.25.

### 41. `runtime.AddCleanup` preferred over `SetFinalizer` (Go 1.24)

**Claude may suggest `SetFinalizer`**: Prefer `runtime.AddCleanup` — multiple cleanups per object, works with interior pointers, no cycle leaks, no delayed freeing. In Go 1.25, cleanups run concurrently.

### 42. `weak` package for weak pointers (Go 1.24)

**Claude may not suggest**: `weak.Pointer[T]` for memory-efficient weak references. Use with `runtime.AddCleanup` and `maphash.Comparable`.

### 43. Cgo `#cgo noescape` / `#cgo nocallback` annotations (Go 1.24)

**Claude may not suggest**: These annotations improve cgo performance by telling the compiler that memory doesn't escape to C or that C doesn't call back to Go.

### 44. `net/http` unencrypted HTTP/2 and `Protocols` field (Go 1.24)

**Claude may not know about**: `Server.Protocols` / `Transport.Protocols` fields. Unencrypted HTTP/2 ("prior knowledge", RFC 9113 §3.3) support — server accepts HTTP/1 and h2c on same port.

### 45. `testing.B.Loop` for benchmarks (Go 1.24)

**Claude may suggest `for range b.N`**: Prefer `for b.Loop() { ... }` — executes once per -count, keeps call params/results alive. Note: in Go 1.26, `B.Loop` no longer prevents inlining.

### 46. `hash/maphash.Comparable` and `WriteComparable` (Go 1.24)

**Claude may not suggest**: These functions hash any comparable value, enabling custom hash maps over any map-key-compatible type.

## How to use this file in other projects

Add to your project's `CLAUDE.md`:

```markdown
## Go Version Notes
This project uses Go 1.27+.
See docs/go/claude-knowledge-corrections.md for
post-training Go changes that Claude's base knowledge gets wrong.
```

Or copy this file directly into your project's docs.
