# Go Best Practices Guide

A practical checklist and reference for writing solid Go code from day one.

## Quick Reference: Commands You Should Know

| Command | What it does | When to run |
|---------|-------------|-------------|
| `gofmt -w .` | Formats all Go files to standard style | Every save (or configure editor to do it) |
| `go vet ./...` | Catches bugs the compiler misses | Before every commit |
| `go fix ./...` | Modernizes code to use newer Go features | After upgrading Go, or periodically |
| `go fix -diff ./...` | Preview what `go fix` would change (without applying) | Before running `go fix` for the first time |
| `go mod tidy` | Cleans up `go.mod` and `go.sum` | After adding/removing imports |
| `go build ./...` | Compiles everything to check for errors | Before committing |
| `go test ./...` | Runs all tests | Before committing |
| `golangci-lint run` | Runs dozens of linters at once | Before committing |

The `./...` pattern means "this package and everything under it." You'll use it constantly.

## Pre-Commit Checklist

Run these in order before committing:

```bash
gofmt -w .
go fix ./...
go vet ./...
go build ./...
go test ./...
go mod tidy
```

If any command produces output or errors, fix them before committing.

## Setting Up a New Project

1. **`go mod init <module-name>`** — creates your `go.mod` file
2. **Run `go mod tidy`** after adding any imports — this generates `go.sum`
3. **Commit both `go.mod` and `go.sum`** — `go.sum` is a lockfile for your dependencies
4. **Set up format-on-save** in your editor (see Editor Setup below)
5. **Install `golangci-lint`** — `brew install golangci-lint`

## Go Fix: Modernizing Your Code (Go 1.26+)

`go fix` was completely rewritten in Go 1.26. It now automatically modernizes your code to use newer language features and idioms. Think of it as an automated code reviewer that knows every Go release.

### How to use it

```bash
# Preview changes first (recommended)
go fix -diff ./...

# Apply changes
go fix ./...

# Run it twice — some fixes unlock further fixes
go fix ./...
go fix ./...
```

Always run from a **clean git state** so you can easily review changes with `git diff`.

### What it fixes

These are called "fixers" or "modernizers." Each one targets a specific pattern:

| Fixer | What it does | Min Go version |
|-------|-------------|----------------|
| `any` | `interface{}` → `any` | 1.18 |
| `minmax` | if/else chains → `min()`/`max()` | 1.21 |
| `rangeint` | `for i := 0; i < n; i++` → `for range n` | 1.22 |
| `forvar` | Removes redundant loop variable re-declarations | 1.22 |
| `stringscut` | `strings.Index` + slicing → `strings.Cut` | 1.18 |
| `stringsbuilder` | String concatenation in loops → `strings.Builder` | 1.10 |
| `fmtappendf` | `[]byte(fmt.Sprintf(...))` → `fmt.Appendf` | 1.19 |
| `mapsloop` | Explicit map loops → `maps` package calls | 1.21 |
| `newexpr` | Custom `newFoo` helpers → `new(value)` | 1.26 |

### Version gating

`go fix` reads your `go` directive in `go.mod` to know which fixes are safe to apply. If your `go.mod` says `go 1.21`, it won't apply fixes that require Go 1.22+. Keep your `go` directive current to get the most out of `go fix`.

### Running specific fixers

```bash
# Only run the 'minmax' fixer
go fix -minmax ./...

# Run everything except 'any'
go fix -any=false ./...

# See all available fixers and what they do
go tool fix help

# Get details on a specific fixer
go tool fix help stringscut
```

### Cross-platform coverage

If your code uses build tags or supports multiple platforms, run `go fix` for each:

```bash
GOOS=linux   GOARCH=amd64 go fix ./...
GOOS=darwin  GOARCH=arm64 go fix ./...
GOOS=windows GOARCH=amd64 go fix ./...
```

## Code Quality Habits

### Always handle errors

This is the single most important Go habit. If a function returns an error, handle it.

```go
// BAD — ignoring the error
data, _ := os.ReadFile("config.json")

// GOOD
data, err := os.ReadFile("config.json")
if err != nil {
    return fmt.Errorf("reading config: %w", err)
}
```

`golangci-lint` has an `errcheck` linter that catches unhandled errors automatically.

### Wrap errors with context

When returning errors, wrap them so you can trace where things went wrong:

```go
// Less helpful
return err

// Much more helpful — tells you what was happening when it failed
return fmt.Errorf("fetching story %d: %w", id, err)
```

### Use `gofmt` — no exceptions

Go has one official style. Don't fight it. Every Go developer uses `gofmt`. This means you never waste time debating formatting.

### Write tests next to your code

Tests for `mcp.go` go in `mcp_test.go` in the same directory. Start simple:

```go
func TestStripHTML(t *testing.T) {
    got := stripHTML("<p>hello <b>world</b></p>")
    want := "hello world"
    if got != want {
        t.Errorf("stripHTML() = %q, want %q", got, want)
    }
}
```

### Keep functions short and focused

If a function is getting long, break it up. Each function should do one thing. This also makes testing easier — you can test small pieces in isolation.

### Name things clearly

Go favors short but descriptive names. Some conventions:
- Receivers: short, 1-2 letters (`s` for server, `c` for client)
- Local variables: short is fine (`i`, `err`, `ctx`)
- Exported functions: descriptive (`FetchStory`, not `FS`)
- Packages: short, lowercase, one word (`html`, `comments`, not `htmlUtils`)

## Editor Setup (Zed)

Add to your Zed settings to format on save and run checks automatically:

```json
{
  "languages": {
    "Go": {
      "format_on_save": "language_server",
      "formatter": "language_server"
    }
  }
}
```

Make sure you have the Go extension installed in Zed, which provides `gopls` (the Go language server). It handles formatting, imports, and inline error checking.

## Useful Tools to Install

```bash
# The essentials
brew install golangci-lint              # Multi-linter runner

# Handy extras
go install golang.org/x/tools/cmd/goimports@latest  # gofmt + import organization
go install golang.org/x/vuln/cmd/govulncheck@latest # Check for known vulnerabilities
```

### govulncheck

Scans your dependencies for known security vulnerabilities:

```bash
govulncheck ./...
```

Run this periodically, especially before releases.

## Project Structure

For a small project (like this one), keeping everything in one package is fine. As it grows, consider splitting by responsibility:

```
myproject/
├── main.go          # Entry point
├── server.go        # Server setup
├── handlers.go      # Request handlers
├── types.go         # Shared types
├── client.go        # External API client
├── client_test.go   # Tests for client
└── go.mod
```

Don't create packages prematurely. The Go proverb is: "A little copying is better than a little dependency."

## Summary: The Go Developer Loop

1. Write code
2. Save → editor auto-formats with `gofmt`
3. `go vet ./...` → catch subtle bugs
4. `go test ./...` → make sure nothing is broken
5. `go fix ./...` → modernize (especially after Go upgrades)
6. `go mod tidy` → clean up dependencies
7. Commit from a clean state
