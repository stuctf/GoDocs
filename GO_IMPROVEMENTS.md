# Claude Code Improvements for Go Workflow

Additions to complement `GO_BEST_PRACTICES.md` with Claude Code-specific automation.

## 1. CLAUDE.md per Go project

Drop a `CLAUDE.md` in your Go project root so Claude Code automatically follows your practices:

```markdown
# Go Project Instructions

## Verification
- Formatting is handled automatically by the PostToolUse hook (gofmt after every edit)
- Run `/go-check` for the full checklist (vet, build, test, mod tidy) before finishing a task

## Error handling
- Never ignore errors with `_`
- Always wrap errors with context: `fmt.Errorf("doing X: %w", err)`
- Prefer `errors.AsType[T](err)` over `errors.As` (Go 1.26+)

## Security
- Use `crypto/rand` not `math/rand` for tokens, keys, and secrets
- Use parameterized queries, never string interpolation for SQL
- Use `os.Root` for filesystem access with user-provided paths — prevents path traversal (Go 1.24+)

## Style
- Use `any` not `interface{}`
- Use `for range n` not `for i := 0; i < n; i++`
- Use `new(expr)` for pointer-to-value (e.g. `p.Age = new(yearsSince(born))`)
- Use `wg.Go(func() { ... })` not manual Add/go/Done (Go 1.25+)
- Use `omitzero` not `omitempty` when omitting zero values in JSON tags — especially for struct types like `time.Time` where `omitempty` doesn't work (Go 1.24+)
- Use `b.Loop()` not `for range b.N` in benchmarks (Go 1.24+)
- Use the stdlib `uuid` package, not `github.com/google/uuid` (Go 1.27+)
- Use `strings.CutLast` / `bytes.CutLast`, not `LastIndex` plus slicing (Go 1.27+)
- Run `go fix ./...` after making changes
```

This saves you from repeating corrections every session.

## 2. Hooks for automatic checks

Claude Code hooks can run your pre-commit checklist automatically after edits. In `~/.claude/settings.json` (or project-level `.claude/settings.json`):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cd $CLAUDE_PROJECT_DIR && gofmt -w ."
          }
        ]
      }
    ]
  }
}
```

This keeps formatting correct after every edit with negligible overhead. Heavy checks (vet, build, test) are deferred to `/go-check` so they don't block every edit.

## 3. Custom slash command for checks (and optional commit)

Create `.claude/skills/go-check/SKILL.md`:

```markdown
---
name: go-check
description: Run the full Go pre-commit checklist and fix any issues found. Pass "commit" to also commit.
disable-model-invocation: true
allowed-tools: Bash, Read, Edit, Write
argument-hint: [commit]
---

Run the full Go pre-commit checklist on the project and fix any issues:

0. Check that the `go` directive in `go.mod` matches the Go version in use — `go fix` is version-gated and won't apply newer fixers if the directive is stale
1. gofmt -w .
2. go fix ./... (run twice — some fixes unlock further fixes)
3. go vet ./...
4. go build ./...
5. go test ./...
6. go mod tidy

If any step fails, fix the issue and re-run from that step.

If called with "commit" ($ARGUMENTS): after all checks pass, commit with a descriptive message.
```

- `/go-check` — runs the full checklist
- `/go-check commit` — runs the checklist, then commits if everything passes

## 4. Reference the best practices doc in CLAUDE.md

Since you already have `docs/go/GO_BEST_PRACTICES.md` and the version-specific changelogs, point Claude Code at them:

```markdown
## Go Version Notes
This project uses Go 1.27+.
See docs/go/claude-knowledge-corrections.md for
post-training Go changes that Claude's base knowledge gets wrong.

## Go Reference Docs
- Read `docs/go/GO_BEST_PRACTICES.md` for coding standards
- Read `docs/go/go1.27-changes.md` for latest Go features
```

The version notes block matches the format recommended by the corrections doc itself, ensuring Claude reads it and corrects its stale training knowledge.

## 5. `golangci-lint` — already covered by Zed

The best practices doc mentions `golangci-lint` but it's not in the pre-commit checklist or Claude Code automation. It doesn't need to be — the Zed golangci-lint extension runs it in real-time on every save, so any issues Claude introduces will surface in the editor automatically.

Enable `gosec` in your `.golangci.yml` to get security-focused static analysis (hardcoded credentials, SQL injection, weak crypto, etc.) as part of the same real-time Zed linting — it's bundled in golangci-lint but off by default.

## 6. Use `tool` directives for project-specific tools

Go 1.24 added `tool` directives in `go.mod`, replacing the old `tools.go` blank-import pattern. For project-specific tools (linters, generators, etc.), prefer tracking them in `go.mod` so they're versioned and reproducible:

```bash
go get -tool golang.org/x/vuln/cmd/govulncheck@latest
go install tool  # installs all tools in go.mod
```

Note: `golangci-lint` specifically recommends binary installation (`brew install`), so use `tool` directives for other project tools rather than for golangci-lint itself.

## 7. Security scanning — run manually, not automated

`govulncheck ./...` scans dependencies for known CVEs. It only matters when dependencies change, not when writing code — so it doesn't belong in hooks or `/go-check`. Run it manually after adding/updating deps, or periodically before releases. Already covered in the best practices doc.

---

**Bottom line:** The best practices doc is solid as a human reference. These Claude Code additions layer on automation so the checks happen *automatically* rather than requiring you to remember to run them. The highest-value additions are the **CLAUDE.md** (so every session starts right) and the **PostToolUse hook** (so issues get caught immediately, not at commit time).
