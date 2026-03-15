# Go 1.25 Release Changes (August 2025)

> Post-training knowledge for Claude (cutoff May 2025).
> Source: https://go.dev/doc/go1.25

## Language Changes

None. The spec editorial change removes the concept of "core types" in favor of dedicated prose. No functional impact.

## Runtime

### Container-aware GOMAXPROCS

GOMAXPROCS default behavior changed:

1. **Linux cgroup awareness**: If the cgroup CPU bandwidth limit is lower than logical CPU count, GOMAXPROCS defaults to the lower limit. In Kubernetes, this corresponds to "CPU limit" (not "CPU requests").
2. **Dynamic updates**: On all OSes, the runtime periodically updates GOMAXPROCS if available CPUs or cgroup limits change.

Both behaviors auto-disable if GOMAXPROCS is set manually (env var or `runtime.GOMAXPROCS()`). Can also be disabled with GODEBUG settings `containermaxprocs=0` and `updatemaxprocs=0`.

New `runtime.SetDefaultGOMAXPROCS()` function re-enables the default behavior after manual override.

### Green Tea GC (experimental)

New garbage collector design. Improves marking/scanning of small objects through better locality and CPU scalability. **10-40% GC overhead reduction** in real-world programs. Enable with `GOEXPERIMENT=greenteagc`. Becomes default in Go 1.26.

### Trace Flight Recorder

`runtime/trace.FlightRecorder` API — lightweight continuous trace recording into an in-memory ring buffer. Call `FlightRecorder.WriteTo` to snapshot the last few seconds when something interesting happens. Configurable via `FlightRecorderConfig`.

### VMA Names on Linux

On Linux with `CONFIG_ANON_VMA_NAME`, the runtime annotates anonymous memory mappings (e.g., `[anon: Go: heap]`). Disable with GODEBUG `decoratemappings=0`.

### Unhandled Panic Output Change

Recovered-and-repanicked output no longer repeats the panic value:
```
// Before: panic: PANIC [recovered]
//           panic: PANIC
// After:  panic: PANIC [recovered, repanicked]
```

## Compiler

### Nil Pointer Bug Fix (BREAKING)

**Fixed a bug from Go 1.21** that incorrectly delayed nil pointer checks. Code like this was silently "working" in Go 1.21-1.24 but now correctly panics:

```go
f, err := os.Open("nonExistent")
name := f.Name()  // should panic if f is nil, now does
if err != nil { return }
```

Fix: move the error check before using the result.

### DWARF5 Support

Compiler/linker now generate DWARF5 debug info. Smaller binaries, faster linking. Disable with `GOEXPERIMENT=nodwarf5`.

### Faster Slices

Compiler allocates slice backing stores on the stack in more situations. May amplify incorrect `unsafe.Pointer` usage. Bisect with `-compile=variablemake`, disable with `-gcflags=all=-d=variablemakehash=n`.

## Tools

### Go Command

- **`go.mod ignore` directive**: Specify directories the go command should ignore for pattern matching (`all`, `./...`) but still included in module zip files.
- **`go doc -http`**: Starts a documentation server and opens browser.
- **`go version -m -json`**: Prints JSON-encoded `runtime/debug.BuildInfo`.
- **Module subdirectory support**: `<meta name="go-import">` now supports `subdir` attribute.
- **`work` package pattern**: Matches all packages in the work modules.
- **Toolchain line**: `go mod` no longer adds a toolchain line when updating the `go` line.

### Vet

New analyzers:
- **`waitgroup`**: Reports misplaced `sync.WaitGroup.Add` calls.
- **`hostport`**: Flags `fmt.Sprintf("%s:%d", host, port)` (IPv6-unfriendly), suggests `net.JoinHostPort`.

## New Packages

### `testing/synctest` (GA)

Graduated from `GOEXPERIMENT=synctest` (Go 1.24) to general availability. `Test` runs a function in an isolated "bubble" with virtualized time. `Wait` waits for all goroutines in the bubble to block. Old API still present under GOEXPERIMENT but removed in 1.26.

### `encoding/json/v2` (experimental)

Enable with `GOEXPERIMENT=jsonv2`. Two new packages:
- `encoding/json/v2` — major revision of `encoding/json`
- `encoding/json/jsontext` — lower-level JSON syntax processing

When enabled, `encoding/json` transparently uses the new implementation. Encoding at parity, **decoding substantially faster**. Marshaling/unmarshaling behavior unchanged, but error text may differ.

## Standard Library Highlights

### `sync`
- **`WaitGroup.Go`**: Combines `wg.Add(1)` + `go func()` + `defer wg.Done()` in one call.

### `net/http`
- **`CrossOriginProtection`**: Built-in CSRF protection using Fetch metadata headers. No tokens/cookies. Supports origin-based and pattern-based bypasses.

### `io/fs`
- **`ReadLinkFS` interface**: Read symbolic links in a filesystem. Implemented by `os.DirFS` and `os.Root.FS`.

### `hash`
- **`Cloner` interface**: Hash implementations can return a copy of their state. All stdlib hashes implement it.
- **`XOF` interface**: For extendable output functions (e.g., SHAKE).

### `reflect`
- **`TypeAssert`**: Convert a `Value` directly to a Go value of given type, avoiding allocations from `Value.Interface()`.

### `crypto`
- **`MessageSigner` interface**: Signers that hash the message themselves. New `SignMessage` function.
- SHA-1, SHA-256, SHA-512 slower on amd64 without AVX2. SHA-1 2x faster with SHA-NI. SHA-3 2x faster on Apple M. RSA keygen 3x faster.

### `crypto/tls`
- **`ConnectionState.CurveID`**: Exposes key exchange mechanism.
- **Encrypted Client Hello**: `Config.GetEncryptedClientHelloKeys` callback.
- SHA-1 signature algorithms disallowed in TLS 1.2 (re-enable with `tlssha1=1`).
- Servers prefer highest supported protocol version.

### `crypto/ecdsa`
- New `ParseRawPrivateKey`, `ParseUncompressedPublicKey`, `PrivateKey.Bytes`, `PublicKey.Bytes`.

### `os`
- Windows: `NewFile` supports async I/O handles (`FILE_FLAG_OVERLAPPED`), with non-blocking I/O and deadline support.
- `Root` type: Many new methods (`Chmod`, `Chown`, `Chtimes`, `Link`, `MkdirAll`, `ReadFile`, `Readlink`, `RemoveAll`, `Rename`, `Symlink`, `WriteFile`).
- `DirFS` and `Root.FS` implement `ReadLinkFS`. `CopyFS` supports symlinks.

### `go/ast`
- `PreorderStack`: Like `Inspect` but provides the stack of enclosing nodes.
- `FilterPackage`, `PackageExports`, `MergePackageFiles` deprecated.

### `go/types`
- `Var.Kind()`: Classifies variable (package-level, receiver, parameter, result, local, struct field).
- `LookupSelection`: Like `LookupFieldOrMethod` but returns `Selection`.

### `go/token`
- `FileSet.AddExistingFiles`: Add existing `File`s to a `FileSet`.

### `testing`
- **`T.Attr` / `B.Attr` / `F.Attr`**: Emit key-value attributes to test log (`=== ATTR TestName key value`).
- **`T.Output` / `B.Output` / `F.Output`**: Returns `io.Writer` to test output stream (like `Log` but no file/line prefix).
- `AllocsPerRun` now panics if parallel tests are running.

### `runtime`
- `AddCleanup` functions now run concurrently and in parallel.
- New GODEBUG `checkfinalizers=1` for diagnosing finalizer/cleanup issues.

### `regexp/syntax`
- `\p{name}` now accepts `Any`, `ASCII`, `Assigned`, `Cn`, `LC`, category aliases like `\p{Letter}`, and case-insensitive lookups ignoring spaces/underscores/hyphens.

### `unique`
- Values reclaimed more eagerly and in parallel. `Handle` chains collected in single GC cycle.

### `unicode`
- New `CategoryAliases` map, new categories `Cn` (unassigned) and `LC` (cased letters). `C` category now includes `Cn`.

## Ports

- **Darwin**: Requires macOS 12 Monterey minimum.
- **AMD64**: `GOAMD64=v3`+ uses fused multiply-add instructions. May change exact floating-point values. Use `float64(a*b)+c` to avoid fusing.
- **Loong64**: Race detector, cgo traceback, internal link mode support.
- **RISC-V**: Plugin build mode. New `GORISCV64=rva23u64` value.
- **Windows/arm (32-bit)**: Last release containing this broken port.
