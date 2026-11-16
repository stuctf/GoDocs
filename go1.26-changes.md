# Go 1.26 Release Changes (February 2026)

> Post-training knowledge for Claude (cutoff May 2025).
> Source: https://go.dev/doc/go1.26

## Language Changes

### `new(expression)` form

The built-in `new` function now accepts an expression (not just a type), creating a pointer to a new variable initialized with the expression's value.

```go
new(int)                // *int → 0 (existing behavior)
new(42)                 // *int → 42 (NEW)
new(yearsSince(born))   // *int → result of yearsSince(born) (NEW)
```

**Motivation**: Optional pointer fields in serialization:
```go
type Person struct {
    Name string
    Age  *int `json:"age"` // nil = unknown
}

// Before: age := yearsSince(born); p.Age = &age
// After:  Age: new(yearsSince(born))

func personJSON(name string, born time.Time) ([]byte, error) {
    return json.Marshal(Person{
        Name: name,
        Age:  new(yearsSince(born)),
    })
}
```

Rules:
- If argument is a type `T`: allocates zero-valued `T` (unchanged)
- If argument is an expression `x`: allocates variable of type-of-x, initialized to x
- Untyped constants convert to default type; untyped booleans convert to `bool`
- `nil` cannot be used as argument

### Self-referential generic type constraints

The restriction that a generic type may not refer to itself in its type parameter list has been **lifted**. F-bounded polymorphism is now possible:

```go
type Adder[A Adder[A]] interface {
    Add(A) A
}

func algo[A Adder[A]](x, y A) A {
    return x.Add(y)
}
```

Previously `Adder[A Adder[A]]` was illegal. This enables patterns like Java's `Comparable<T extends Comparable<T>>`.

## Runtime

### Green Tea GC (now default)

The experimental GC from Go 1.25 is now **enabled by default**. 10-40% GC overhead reduction. On newer amd64 CPUs (Intel Ice Lake / AMD Zen 4+), uses vector instructions for ~10% further improvement.

Disable with `GOEXPERIMENT=nogreenteagc`. Opt-out expected to be removed in Go 1.27. (**Update:** it was not removed; `nogreenteagc` still works in Go 1.27.)

### Faster cgo Calls

Baseline runtime overhead of cgo calls reduced by ~30%.

### Heap Base Address Randomization

On 64-bit platforms, runtime randomizes the heap base address at startup. Security hardening against address prediction exploits via cgo. Disable with `GOEXPERIMENT=norandomizedheapbase64`.

### Goroutine Leak Profile (experimental)

New `goroutineleak` profile type. Enable with `GOEXPERIMENT=goroutineleakprofile`.

Detects goroutines blocked on concurrency primitives (channels, `sync.Mutex`, `sync.Cond`, etc.) that **cannot possibly be unblocked** — the primitive is unreachable from any runnable goroutine. Uses GC reachability analysis.

```go
// Classic leak example: early return with unbuffered channel
func processWorkItems(ws []workItem) ([]workResult, error) {
    ch := make(chan result)
    for _, w := range ws {
        go func() {
            res, err := processWorkItem(w)
            ch <- result{res, err}  // leaks if processWorkItems returns early
        }()
    }
    for range len(ws) {
        r := <-ch
        if r.err != nil {
            return nil, r.err  // early return → remaining goroutines leak
        }
        // ...
    }
    // ...
}
```

Also available as `net/http/pprof` endpoint `/debug/pprof/goroutineleak`. Zero overhead when not in use. Expected default in Go 1.27.

## Compiler

- Slice backing store stack allocation expanded (same as 1.25 improvement, continuing).
- Bisect with `-compile=variablemake`, disable with `-gcflags=all=-d=variablemakehash=n`.

## Linker

- Windows/arm64: Internal linking mode for cgo programs.
- `.gosymtab` section removed (was always empty).
- `.gopclntab` section no longer contains relocations; moved to rodata segment on relro platforms.
- `moduledata` in its own `.go.module` section.
- ELF sections now sorted by address in section header.

## Tools

### `go fix` Revamped

Complete rewrite. Now the home of Go's **modernizers**:
- Dozens of fixers to update code to latest Go idioms and APIs
- **Source-level inliner** via `//go:fix inline` directives for automating API migrations
- Built on the same analysis framework as `go vet`
- Historical fixers (all obsolete) removed

### `go mod init` Version Policy

`go mod init` with toolchain `1.N.X` now creates `go.mod` with `go 1.(N-1).0`. Pre-release versions use `go 1.(N-2).0`. Encourages compatibility with currently supported Go versions.

### Other

- `cmd/doc` and `go tool doc` deleted. Use `go doc` directly.
- `pprof` web UI defaults to flame graph view.

## Bootstrap

Go 1.26 requires Go 1.24.6+ for bootstrap. Go 1.28 will require Go 1.26+.

## New Packages

### `crypto/hpke`

Hybrid Public Key Encryption (RFC 9180) with post-quantum hybrid KEM support.

### `simd/archsimd` (experimental)

Enable with `GOEXPERIMENT=simd`. Architecture-specific SIMD operations (amd64 only for now). 128/256/512-bit vector types:

```go
// Types like:
Int8x16, Float64x8
// Operations like:
vec.Add(other)
```

Portable high-level SIMD package planned for future.

### `runtime/secret` (experimental)

Enable with `GOEXPERIMENT=runtimesecret`. Secure erasure of temporaries (registers, stack, heap) used in cryptographic code. For forward secrecy. amd64/arm64 Linux only.

### `crypto/mlkem/mlkemtest`

Derandomized ML-KEM encapsulation for known-answer tests.

## Standard Library Highlights

### `errors`
- **`AsType[T]`**: Generic, type-safe version of `As`. Faster and easier:
  ```go
  // Before:
  var pathErr *fs.PathError
  if errors.As(err, &pathErr) { ... }
  // After:
  if pathErr, ok := errors.AsType[*fs.PathError](err); ok { ... }
  ```

### `io`
- **`ReadAll` optimization**: ~2x faster, ~50% less memory. Returns minimally sized final slice.

### `log/slog`
- **`NewMultiHandler`**: Fan-out handler that invokes all given handlers. `Enabled` returns true if any sub-handler is enabled.

### `reflect`
- **Iterator methods on `Type`**: `Fields()`, `Methods()`, `Ins()`, `Outs()` return iterators.
- **Iterator methods on `Value`**: `Fields()`, `Methods()` yield type info + value pairs.

### `testing`
- **`T.ArtifactDir` / `B.ArtifactDir` / `F.ArtifactDir`**: Returns directory for test output files. With `-artifacts` flag, persists after test. Emits `=== ARTIFACTS TestName /path`.
- **`B.Loop` fix**: No longer prevents inlining in loop body. All benchmarks can now safely use `B.Loop` style.

### `net/http`
- `HTTP2Config.StrictMaxConcurrentRequests`: Control whether new connections open when stream limit exceeded.
- `Transport.NewClientConn`: Returns a client connection for custom connection management.
- `Client` now scopes cookies to `Request.Host` when available.
- `ServeMux` trailing slash redirects now use 307 instead of 301.

### `net/http/httptest`
- `Server.Client()` now redirects `example.com` requests to the test server.

### `net/http/httputil`
- `ReverseProxy.Director` **deprecated** in favor of `Rewrite`. Director is fundamentally unsafe (hop-by-hop header stripping attack).

### `net/netip`
- `Prefix.Compare`: Compares two prefixes.

### `net/url`
- `Parse` rejects malformed URLs with colons in host (e.g., `http://::1/`). Bracketed IPv6 still accepted. GODEBUG `urlstrictcolons=0` restores old behavior.

### `os`
- `Process.WithHandle`: Access internal process handle (pidfd on Linux, Handle on Windows).
- Windows: `OpenFile` flag accepts Windows-specific file flags (`FILE_FLAG_OVERLAPPED`, `FILE_FLAG_SEQUENTIAL_SCAN`).

### `os/signal`
- `NotifyContext` now cancels with `CancelCauseFunc` and error indicating which signal was received.

### `crypto/tls`
- **Post-quantum default**: `SecP256r1MLKEM768` and `SecP384r1MLKEM1024` hybrid key exchanges enabled by default. Disable with `tlssecpmlkem=0`.
- `ClientHelloInfo.HelloRetryRequest` and `ConnectionState.HelloRetryRequest` fields.
- GODEBUG removals scheduled for Go 1.27: `tlsunsafeekm`, `tlsrsakex`, `tls10server`, `tls3des`, `x509keypairleaf`.

### `crypto/rsa`
- `EncryptOAEPWithOptions`: Different hash functions for OAEP padding and MGF1.
- PKCS#1 v1.5 encryption padding deprecated.
- Random parameters to `GenerateKey`, `EncryptPKCS1v15`, etc. now ignored (always use secure random).

### `crypto/sha3`
- Zero value of `SHA3` is usable SHA3-256; zero value of `SHAKE` is usable SHAKE256.

### `crypto/subtle`
- `WithDataIndependentTiming` no longer locks to OS thread. Spawned goroutines inherit DIT. cgo integration improved.

### `crypto/x509`
- `ExtKeyUsage` and `KeyUsage` have `String()` methods.
- `ExtKeyUsage.OID()` method.
- `OIDFromASN1OID` function.

### `fmt`
- `fmt.Errorf("x")` now allocates less, matching `errors.New("x")`.

### `go/ast`
- `ParseDirective`: Parses directive comments (`//go:generate` etc.).
- `BasicLit.ValueEnd`: Precise end position for multi-line raw string literals.

### `go/types`
- `gotypesalias` GODEBUG removed in Go 1.27. `go/types` will always produce `Alias` for type aliases.

### `image/jpeg`
- Encoder and decoder **replaced** with faster, more accurate implementations. Not bit-for-bit compatible.

### `bytes`
- `Buffer.Peek(n)`: Returns next n bytes without advancing.

### `runtime/metrics`
- New scheduler metrics: goroutine state counts (`/sched/goroutines` prefix), OS thread count, total goroutines created.

### `testing/cryptotest`
- `SetGlobalRandom`: Configure deterministic crypto randomness for testing.

### `time`
- `asynctimerchan` GODEBUG removed in Go 1.27. Timers always use unbuffered channels.

## Ports

- **Darwin**: Last release for macOS 12 Monterey. Go 1.27 requires macOS 13+.
- **Windows/arm (32-bit)**: Removed.
- **FreeBSD/riscv64**: Marked broken.
- **Linux/riscv64**: Race detector support.
- **S390X**: Register-based function argument passing.
- **WebAssembly**: Unconditional sign extension and non-trapping float-to-int. `signext`/`satconv` GOWASM settings ignored. Much smaller heap memory usage for <16 MiB heaps.
- **PowerPC**: Last release for ELFv1 ABI on big-endian ppc64. Go 1.27 switches to ELFv2.
