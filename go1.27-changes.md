# Go 1.27 Release Changes (August 2026)

> Post-training knowledge for Claude (cutoff June 2026).
> Source: https://go.dev/doc/go1.27

## Language Changes

### Generic methods

A method declaration may now declare **its own type parameters**, in addition to any the receiver type has. This was one of the most requested Go features. Before this change, generic helpers for a type had to be package-level functions.

```go
type Set[T comparable] struct{ m map[T]struct{} }

// NEW: method with its own type parameter U
func (s *Set[T]) Map[U comparable](f func(T) U) *Set[U] {
    out := &Set[U]{m: make(map[U]struct{}, len(s.m))}
    for v := range s.m {
        out.m[f(v)] = struct{}{}
    }
    return out
}

names := ids.Map(strconv.Itoa) // U inferred as string
```

Real stdlib example: `math/rand/v2` now has the method `(*Rand) N[Int intType](Int) Int`, which matches the top-level `rand.N`:

```go
r := rand.New(rand.NewPCG(1, 2))
d := r.N(10 * time.Second) // time.Duration in [0, 10s)
```

Restrictions:
- **Interface methods may not declare type parameters.**
- **Generic methods cannot implement interface methods.** A type's generic method does not count toward satisfying an interface.

### Struct literal keys may name promoted fields

A key in a struct literal can now be any valid field selector for the struct type, not just a top-level field name. In practice this means you can set **promoted fields from embedded structs** directly:

```go
type Object  struct{ name, color string }
type Point3D struct{ Object; x, y, z float64 }

// Before: Point3D{Object: Object{name: "a"}, x: 1}
// After:
p := Point3D{name: "a", x: 1}
```

Rules:
- Embedded fields traversed to reach the selected field must **not be pointer types**.
- You can't set a promoted field and also the embedded struct that contains it (`Point3D{Object: o, name: "a"}` is an error).

### Generalized function type inference

Type inference for generic functions now applies in **all contexts** where a generic function is assigned to (or converted to) a value of a matching function type. Go 1.21 handled only some of these, such as plain variable assignment and return values. If you used to write explicit instantiations like `Identity[int]` because inference failed, try dropping them.

## Tools

### Response files

`compile`, `link`, `asm`, `cgo`, `cover`, and `pack` now accept `@file` response files: whitespace-separated arguments with quoting, escapes, and backslash-newline continuation. The format is GCC-compatible.

### `go` command

- **`bzr` support removed.** Modules hosted on Bazaar servers can no longer be fetched directly.
- **Removed `GODEBUG` settings are tolerated** in `go.mod` `godebug` entries and `//go:debug` comments, *if* set to the final default value. Setting a removed `GODEBUG` to an old value makes the build **fail**.

### `go test`

- **`stdversion` vet check now runs by default.** It reports use of stdlib symbols that are newer than the `go` directive in `go.mod` (or the file's build tags).
- `go test -json` output lines may include a new `"OutputType"` field (`"error"`, `"error-continue"`, `"frame"`).

### `go doc`

- Supports `package@version`: `go doc example.com/pkg@v1.2.3`.
- New `-ex` flag lists executable examples. Passing an example name (`go doc bytes.ExampleBuffer`) prints its source and comments.

### `go fix`

- New modernizers: `atomictypes`, `embedlit`, `slicesbackward`, `unsafefuncs`.
- `fmtappendf` analyzer **removed** for stylistic reasons.
- `waitgroup` analyzer **renamed** to `waitgroupgo`.

### `go mod tidy`

For modules with `go 1.27`+, `go mod tidy` **merges duplicate `require` blocks**, leaving at most two: one for direct dependencies and one for indirect ones. Comments attached to dependencies are kept.

### `gofmt`

Fixed column-alignment logic that varied slightly across CPU architectures. **Running Go 1.27 `gofmt` on already-formatted code may cause minor whitespace diffs** in a few files.

### `go vet`

The `printf` analyzer now flags `%w` with a **pointer to a type that itself implements `error`**:

```go
type MyErr struct{}
func (MyErr) Error() string { return "my err" }

p := &MyErr{}
fmt.Errorf("op failed: %w", p)
// vet: %w wants operand of error type MyErr, not pointer type *MyErr (defeats errors.Is)
```

This is usually a latent bug: `errors.As`/`errors.AsType` and type assertions for `MyErr` won't match the `*MyErr` you wrapped.

### `go tool trace`

`-http=:6060` now listens **on localhost only** (matching `go tool pprof`). Use `-http=0.0.0.0:6060` to listen on all addresses.

## Runtime

### Goroutine leak profile (now GA)

The `goroutineleak` profile, experimental in Go 1.26, is now **generally available** in `runtime/pprof` and at the `net/http/pprof` endpoint `/debug/pprof/goroutineleak`. No `GOEXPERIMENT` is needed; the `goroutineleakprofile` experiment has been deleted.

It detects goroutines blocked on channels, `sync.Mutex`, `sync.Cond`, etc. when the primitive is unreachable from any runnable goroutine, so the goroutine can never wake up. It **may miss** leaks where the primitive is reachable through global variables or the locals of runnable goroutines. See `go1.26-changes.md` for an example of a leak.

### Faster small allocations

The compiler emits calls to size-specialized allocation routines. Small allocations (<80 bytes) are up to **30% cheaper**, which works out to about 1% overall in allocation-heavy programs. Binaries grow by about 60 KB. Disable with `GOEXPERIMENT=nosizespecializedmalloc`; this opt-out is expected to be removed in Go 1.28.

### Goroutine labels in tracebacks

For modules with `go 1.27`+, tracebacks include `runtime/pprof` goroutine labels in the header line. Disable with `GODEBUG=tracebacklabels=0`. This opt-out will be kept indefinitely because labels may contain sensitive data.

### Green Tea GC opt-out still available

The Go 1.26 notes said the `GOEXPERIMENT=nogreenteagc` opt-out would likely be removed in Go 1.27. **It wasn't.** The Go 1.27 release notes don't mention it, and builds with `nogreenteagc` still succeed on Go 1.27.4.

### `asynctimerchan` removed

The `asynctimerchan` GODEBUG has been **removed permanently**. Channels from package `time` are always unbuffered (synchronous).

## Compiler

- Relative filenames in `//line` / `/*line*/` directives now resolve **against the directory of the file containing the directive**, matching `go/scanner`.
- **Simpler closure names**: a function literal gets the same name whether or not its enclosing function is inlined, and identical instances may share code. Tests that check closure symbol names may need updating. Code that (incorrectly) compares function code pointers may now see equal pointers more often.

## Linker

- macOS: new `-macos` and `-macsdk` flags set the `LC_BUILD_VERSION` OS/SDK versions. The defaults are macOS 13.0.0 and SDK 26.2.0.

## New Packages

### `encoding/json/v2` and `encoding/json/jsontext` (GA)

These were experimental in Go 1.25 (`GOEXPERIMENT=jsonv2`) and are **now available without any experiment flag**.

- **`encoding/json/v2`**: `Marshal`, `MarshalWrite`, `MarshalEncode`, `Unmarshal`, `UnmarshalRead`, `UnmarshalDecode`, all taking variadic `Options`.
- **`encoding/json/jsontext`**: low-level, streaming, syntax-only JSON `Encoder`/`Decoder` over `Token` and `Value`.

```go
import "encoding/json/v2"

err := json.UnmarshalRead(r.Body, &req,
    json.RejectUnknownMembers(true),
    json.MatchCaseInsensitiveNames(true),
)

b, err := json.Marshal(v, json.Deterministic(true), json.FormatNilSliceAsNull(true))
```

v2 has **stricter defaults** than v1: it rejects invalid UTF-8 and duplicate object member names. Other notable options: `OmitZeroStructFields`, `StringifyNumbers`, `FormatNilMapAsNull`, `WithMarshalers`/`WithUnmarshalers` (with `MarshalFunc`, `MarshalToFunc`, `UnmarshalFunc`, `UnmarshalFromFunc`), `JoinOptions`, `GetOption`.

**`encoding/json` (v1) is now backed by the v2 implementation.** Behavior is preserved, but error message text may differ. Unmarshal is **significantly faster**; marshal is about the same speed. v1 gains options to run v2 with v1 semantics (`json.DefaultOptionsV1()`, `OmitEmptyWithLegacySemantics`, `FormatDurationAsNano`, `MatchCaseSensitiveDelimiter`, etc.), so you can migrate gradually. v1 stays supported and migrating is **not required**. Opt out with `GOEXPERIMENT=nojsonv2`; this opt-out is expected to be removed in a future release.

**API changes since the Go 1.25 experiment** (important if you see older v2 code):
- The `inline` struct tag option was **renamed to `embed`**.
- **Removed**: the `format` tag option, the `unknown` tag option, the `DiscardUnknownMembers` option, and the `SkipFunc` sentinel error.
- `string` tag option and `MatchCaseInsensitiveNames` behavior changed.
- `jsontext` numeric `Token` accessors now **also return an error**.

### `crypto/mldsa`

Post-quantum ML-DSA signatures (FIPS 204).

```go
import "crypto/mldsa"

sk, err := mldsa.GenerateKey(mldsa.MLDSA65())
sig, err := sk.Sign(nil, msg, &mldsa.Options{Context: "my-app"})
err = mldsa.Verify(sk.PublicKey(), msg, sig, &mldsa.Options{Context: "my-app"})
```

- Parameter sets: `MLDSA44()`, `MLDSA65()`, `MLDSA87()`.
- `NewPrivateKey` / `NewPublicKey` parse from bytes. `SignDeterministic` is also available.
- `crypto/x509` supports ML-DSA keys and signatures (`x509.MLDSA` public key algorithm; `MLDSA44`/`65`/`87` signature algorithms).
- `crypto/tls` supports ML-DSA in TLS 1.3 (`tls.MLDSA44`, `tls.MLDSA65`, `tls.MLDSA87` `SignatureScheme`s).
- `crypto.MLDSAMu` hash value for External μ ML-DSA signing.

### `uuid`

**UUIDs are now in the standard library.** For new code, use this package instead of `github.com/google/uuid`.

```go
import "uuid"

id := uuid.New()          // currently equivalent to NewV4 (random)
v7 := uuid.NewV7()        // time-ordered; good for DB primary keys
u, err := uuid.Parse("f47ac10b-58cc-4372-a567-0e02b2c3d479")
s := id.String()
```

- `type UUID [16]byte`, which is comparable and usable as a map key.
- `New`, `NewV4`, `NewV7`, `Parse`, `MustParse`, `Nil()`, `Max()`, and `UUID.Compare`.
- Implements `encoding.TextMarshaler`/`TextUnmarshaler`/`TextAppender`, so it works in JSON and similar formats.
- `NewV7` returns increasing values unless the system clock moves backwards.

### `simd` (experimental)

Enable with `GOEXPERIMENT=simd`. A **portable, vector-size-agnostic** SIMD package for all architectures, with types like `Int8s` and `Float32s` of unspecified width. It uses hardware instructions when available and emulates them otherwise.

### `simd/archsimd` (still experimental)

Also behind `GOEXPERIMENT=simd`. The amd64 API has been revised, and there is new support for **arm64 Neon (128-bit)** and **WebAssembly 128-bit SIMD**. The API is architecture-specific and not yet stable.

## Standard Library Highlights

### `strings` / `bytes`
- **`CutLast(s, sep) (before, after string, found bool)`**: like `Cut`, but splits around the **last** occurrence. It replaces many `LastIndex` patterns:
  ```go
  // Before:
  i := strings.LastIndex(path, "/"); dir, file := path[:i], path[i+1:]
  // After:
  dir, file, ok := strings.CutLast(path, "/")
  ```

### `hash/maphash`
- **`Hasher[T]` interface** (`Hash(*maphash.Hash, T)`, `Equal(T, T) bool`): the contract for future generic hash tables and Bloom filters.
- **`ComparableHasher[T comparable]`**: a ready-made `Hasher` that uses `==`.

### `math/big`
- **`(z *Int) Divide(x, y, r *Int, mode RoundingMode) (*Int, *Int)`**: quotient (into `z`) and remainder (into `r`) with rounding modes `big.Trunc`, `big.Floor`, `big.Round`, `big.Ceil`:
  ```go
  q, r := new(big.Int).Divide(big.NewInt(-7), big.NewInt(2), new(big.Int), big.Floor) // q=-4, r=1
  ```

### `math/rand/v2`
- **`(*Rand).N`** generic method, matching top-level `rand.N`.

### `net/http`
- **HTTP/1 `Response.Body.Close()` now drains unread content** (up to a conservative limit) so connections get reused more often. This is usually a no-op or a speedup. Programs that bypass idle-connection limits (`MaxIdleConns: 0`, a new `Client` per request) may see a slowdown; fix the configuration, or set `Transport.DisableKeepAlives`.
- **HTTP/2 server honors RFC 9218 client priority signals**. Set `Server.DisableClientPriority = true` for the old round-robin behavior.
- **`Server.MaxHeaderValueCount`** limits how many header values a server accepts (default `DefaultMaxHeaderValueCount` = 500).
- TLS ALPN negotiation works on user-provided `net.Conn`s that implement `ConnectionState() tls.ConnectionState`.

### `net/http/httptest`
- **`NewTestServer(t, handler)`**: a test server on an **in-memory fake network**, designed for use with `testing/synctest`.

### `testing/synctest`
- **`Sleep(d)`**: combines `time.Sleep` and `synctest.Wait`.

### `net/url`
- **`URL.Clone()`** and **`Values.Clone()`**: deep copies.

### `net`
- `UnixConn` reads return **bare `io.EOF`** instead of wrapping it in `*net.OpError`.

### `database/sql`
- **`ConvertAssign`**: exposes `Rows.Scan` conversions to drivers.
- **`driver.RowsColumnScanner`**: drivers can scan directly into user destinations.

### `crypto/tls`
- **`Config.Rand` deprecated.** For deterministic tests, use `testing/cryptotest.SetGlobalRandom`.
- `MLKEM1024` key exchange is supported (opt in via `Config.CurvePreferences`).
- Post-quantum hybrids listed explicitly in `CurvePreferences` now work even with `tlsmlkem=0` / `tlssecpmlkem=0`. Those GODEBUGs only affect the default set.
- `ConnectionState.LocalCertificate`: the certificate chain you presented.
- `QUICConfig.ClientHelloInfoConn`: the `net.Conn` used for `ClientHelloInfo.Conn` in QUIC server handshakes.
- **GODEBUGs removed permanently**: `tlsunsafeekm`, `tlsrsakex`, `tls3des`, `tls10server`, `x509keypairleaf`. (`tlskyber` was removed back in Go 1.24, which is now documented.)

### `crypto/ecdsa`
- `PrivateKey.Sign` checks the hash length when given non-nil `SignerOpts`.

### `crypto/x509`
- **`SystemCertPool` respects `SSL_CERT_FILE` / `SSL_CERT_DIR` on Windows and macOS.** When they are set, it uses the Go verifier instead of the platform APIs. Disable with `GODEBUG=x509sslcertoverrideplatform=0`.
- `RawSignatureAlgorithm` field on `Certificate`, `CertificateRequest`, `RevocationList`.
- Wider range of `pkix.Name` attribute value types; unknown types parse into `asn1.RawValue`.

### `crypto/x509/pkix`
- `Name.String` renders string-valued attributes with unknown OIDs as strings (previously hex DER).

### `runtime/secret` (experimental)
- Goroutines started while in secret mode also run in secret mode.

### `compress/flate`
- **Faster compression**, but **output bytes may differ** from Go 1.26. This also affects `compress/gzip`, `compress/zlib`, `archive/zip`, and `image/png`. Don't write tests that compare compressed bytes exactly.

### `go/types`
- `Hasher` / `HasherIgnoreTags`: `maphash.Hasher` implementations for `Type` (respecting `Identical`).
- `String()` methods on `TypeList`, `TypeParamList`, `Instance`.
- **`gotypesalias` GODEBUG removed.** `go/types` always produces `Alias` nodes.

### `go/scanner`, `go/token`, `go/constant`
- `Scanner.End()` gives a token's end position. `token.File` has `String()`. `constant.StringLen` returns a string `Value`'s length without fully constructing it.

### `unicode`
- Upgraded from **Unicode 15 to Unicode 17** (`unicode.Version == "17.0.0"`), with new scripts such as `Garay`, `Sunuwar`, `Todhri`, and `Kirat_Rai`.

### `syscall`
- Plan 9: `Errno` type defined (never returned there; exists for portable code).

## GODEBUG Removals Summary

Removed permanently in Go 1.27: `asynctimerchan`, `gotypesalias`, `tlsunsafeekm`, `tlsrsakex`, `tls3des`, `tls10server`, `x509keypairleaf`. The `go` command accepts them only when set to their final default value.

## Ports

- **Darwin**: **Requires macOS 13 Ventura or later.**
- **PowerPC (linux/ppc64, big-endian)**: Now uses the **ELFv2 ABI**, which requires Linux 3.13+ (RHEL7's 3.10 kernel has it backported). cgo, PIE, and external linking are now supported. Pure-Go programs are still statically linked by default.
