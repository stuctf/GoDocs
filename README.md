# GoDocs — Post-Training Go Knowledge for Claude Code

Claude's training data has a cutoff (currently May 2025). That means it doesn't know about Go 1.25 (Aug 2025), Go 1.26 (Feb 2026), or Go 1.27 (Aug 2026) at all, and its knowledge of Go 1.24 (Feb 2025) can be fuzzy. Without corrections, Claude will:

- Suggest outdated patterns when modern replacements exist (`errors.As` instead of `errors.AsType[T]`, manual `WaitGroup` instead of `wg.Go`, `github.com/google/uuid` instead of the stdlib `uuid` package)
- Miss important language changes (`new(expr)`, self-referential generics, generic methods)
- Use insecure defaults it learned from older code (weak crypto, `math/rand` for tokens, string-interpolated SQL)
- Not know about the new Green Tea GC, container-aware GOMAXPROCS, post-quantum TLS and ML-DSA signatures, `encoding/json/v2`, or dozens of other stdlib improvements

These documents fix that. Drop them into your Go project and point Claude Code at them so every session starts with accurate, up-to-date knowledge.

## What's Included

| File | Purpose |
|------|---------|
| `claude-knowledge-corrections.md` | **Start here if you only grab one file.** Lists 46 specific things Claude gets wrong or doesn't know about in Go 1.24–1.27, with corrections and code examples. |
| `go1.24-changes.md` | Comprehensive Go 1.24 release notes — `os.Root`, `tool` directives, `omitzero`, Swiss Tables, crypto improvements, `testing.B.Loop`, and more. |
| `go1.25-changes.md` | Comprehensive Go 1.25 release notes — container-aware GOMAXPROCS, `sync.WaitGroup.Go`, `testing/synctest` GA, `encoding/json/v2`, CSRF protection, and more. |
| `go1.26-changes.md` | Comprehensive Go 1.26 release notes — `new(expr)`, self-referential generics, Green Tea GC, `errors.AsType`, `go fix` modernizers, post-quantum TLS, and more. |
| `go1.27-changes.md` | Comprehensive Go 1.27 release notes — generic methods, `uuid` package, `encoding/json/v2` GA, `crypto/mldsa`, goroutine leak profile GA, `strings.CutLast`, and more. |
| `GO_BEST_PRACTICES.md` | A practical checklist for writing solid Go — pre-commit checks, error handling, `go fix`, editor setup, project structure. Good for humans and Claude alike. |
| `GO_IMPROVEMENTS.md` | How to operationalize all of the above with Claude Code — `CLAUDE.md` templates, hooks, custom slash commands, and automation tips. |

## Quick Start

### 1. Copy the docs into your project

```bash
# From your Go project root:
mkdir -p docs/go

# Option A: Clone and copy
git clone https://github.com/stuctf/GoDocs.git /tmp/GoDocs
cp /tmp/GoDocs/*.md docs/go/
rm docs/go/README.md  # you don't need this file in your project

# Option B: Download individual files with curl
curl -LO https://raw.githubusercontent.com/stuctf/GoDocs/main/claude-knowledge-corrections.md --output-dir docs/go/
curl -LO https://raw.githubusercontent.com/stuctf/GoDocs/main/GO_BEST_PRACTICES.md --output-dir docs/go/
# ... etc
```

### 2. Reference them in your project's CLAUDE.md

Add this to your project's `CLAUDE.md` (create one if you don't have it):

```markdown
## Go Version Notes
This project uses Go 1.27+.
Read docs/go/claude-knowledge-corrections.md for post-training Go changes
that Claude's base knowledge gets wrong.

## Go Reference Docs
- Read docs/go/GO_BEST_PRACTICES.md for coding standards
- Read docs/go/go1.27-changes.md for latest Go features
```

Adjust the Go version to match your project.

### 3. (Optional) Set up automation

See `GO_IMPROVEMENTS.md` for how to set up:

- **A `gofmt` hook** that auto-formats after every Claude Code edit
- **A `/go-check` slash command** that runs the full pre-commit checklist (vet, build, test, mod tidy)
- **CLAUDE.md templates** with security and style rules baked in

## Why Security Defaults Matter

Claude learned from a massive corpus of Go code, and a lot of that code uses insecure patterns that were common (or at least tolerated) for years. These docs push Claude toward safer defaults:

- **`crypto/rand` over `math/rand`** for tokens, keys, and secrets
- **`os.Root`** (Go 1.24+) for filesystem access with user-provided paths — prevents path traversal and symlink escape
- **Parameterized SQL queries** instead of string interpolation
- **`omitzero` over `omitempty`** where `omitempty` silently does the wrong thing (e.g., `time.Time`)
- **Post-quantum TLS** awareness (enabled by default in Go 1.26)
- **`crypto/rand.Text`** (Go 1.24+) for generating random strings — no more hand-rolled token generators
- **`NewGCMWithRandomNonce`** (Go 1.24+) — eliminates nonce-reuse bugs in AES-GCM

## Keeping Up to Date

These docs cover Go 1.24 through Go 1.27. When Go 1.28 ships (expected February 2027), new docs will be needed. Contributions welcome.

## License

These documents are provided as-is for use with Claude Code and other AI coding assistants. Use them however you like.
