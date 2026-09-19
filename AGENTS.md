# AGENTS.md

This file provides guidance to coding agents (Claude Code, Codex, Grok) when working with code in this repository.

## Project Overview

**gogcli** is a single Go CLI (`gog`) providing unified access to Google Workspace and personal Google account APIs: Gmail, Calendar, Drive, Docs, Slides, Sheets, Contacts, Tasks, Classroom, Chat, People, Forms, Apps Script, Keep, and Groups. It supports multi-account OAuth2, service accounts, and secure keyring credential storage.

## Build & Development Commands

```bash
make                # Build binary to bin/gog
make tools          # Install pinned dev tools (.tools/: gofumpt, goimports, golangci-lint)
make fmt            # Format code (goimports + gofumpt)
make fmt-check      # Format check (CI gate; fails if diff exists)
make lint           # Run golangci-lint
make test           # Run all unit tests
make ci             # Full CI gate: pnpm-gate, fmt-check, lint, test
make gog -- --help  # Build + run with flags
```

Optional: `pnpm gog …` builds and runs in one step.

### Running a Single Test

```bash
go test -v ./internal/cmd -run TestGmailSearch
go test -v ./internal/googleapi -run TestClient
```

### Integration Tests (require live OAuth credentials)

```bash
GOG_IT_ACCOUNT=you@gmail.com go test -tags=integration ./internal/integration
scripts/live-test.sh --account you@gmail.com   # End-to-end smoke tests
```

Requires OAuth client credentials plus a stored refresh token in your keyring; these tests are local-only.

### Email Tracking Worker (Node/pnpm)

```bash
make worker-ci   # Lint, build, test internal/tracking/worker
```

### Git Hooks

```bash
lefthook install   # Enable pre-commit and pre-push checks (one-time setup)
```

## Architecture

```
cmd/gog/            CLI entrypoint (main.go)
internal/
  cmd/              All command implementations (~400 files, kong struct-based)
  googleapi/        Google API client wrappers, retry, circuit breaker
  googleauth/       OAuth2 flows (browser/manual/remote), service accounts, scope matrix
  config/           JSON5 config file, account aliases, client mapping
  secrets/          Keyring abstraction (macOS Keychain, encrypted file, Linux Secret Service)
  authclient/       Multi-client OAuth credential resolution
  outfmt/           Output formatting: JSON, plain TSV, human tables (TTY-aware)
  input/            Interactive prompts, readline
  ui/               Colors, progress messages (to stderr)
  errfmt/           Error formatting
  timeparse/        Date/time parsing (relative dates: today, +7d, friday, ISO8601)
  tracking/         Email open tracking (Cloudflare Worker integration)
  integration/      Build-tagged integration tests
docs/               Specs, auth, releasing, date formats
scripts/            live-test.sh, release.sh, gen-auth-services-md.go, gog.mjs
```

### Command Pattern (Kong Framework)

Commands are nested structs with `Run(ctx context.Context, flags *RootFlags) error` methods:

```go
type GmailSearchCmd struct {
    Query []string `arg:"" help:"Search query"`
    Max   int64    `name:"max" help:"Max results" default:"10"`
}
func (c *GmailSearchCmd) Run(ctx context.Context, flags *RootFlags) error { ... }
```

### Service Construction (Mockable)

```go
var newGmailService = googleapi.NewGmail  // Package-level var, overridable in tests
svc, err := newGmailService(ctx, account)
```

### Output Convention

- **stdout**: Parseable data only (JSON `--json`, TSV `--plain`, or human tables)
- **stderr**: Progress, hints, errors, warnings
- Colors auto-detect TTY; respect `NO_COLOR`

## Coding Conventions

- Formatting: `make fmt` (goimports local prefix `github.com/steipete/gogcli` + gofumpt)
  > ⚠️ **Known conflict (CLAUDE.md vs AGENTS.md, unresolved as of merge)**: the AGENTS.md
  > version of this file previously stated the prefix as `github.com/openclaw/gogcli`,
  > which matches the current `module` line in `go.mod`. CLAUDE.md's `steipete` prefix
  > looks stale relative to `go.mod`, but per merge rule the CLAUDE.md wording is kept
  > here; verify against `go.mod` before relying on it.
- Gmail labels: IDs are case-sensitive opaque tokens; only case-fold names for lookup
- Commits: Conventional Commits format (e.g. `feat(cli): add --verbose to send`)
- Follow Conventional Commits + action-oriented subjects (e.g. `feat(cli): add --verbose to send`)
- Group related changes; avoid bundling unrelated refactors
- Unit tests: stdlib `testing` + `httptest` for HTTP mocking; test files next to source
- Headless keyring: set `GOG_KEYRING_BACKEND=file` + `GOG_KEYRING_PASSWORD` for CI/serverless; never commit OAuth credential JSON files or tokens

## Commit & Pull Request Guidelines

- Create commits with `committer "<msg>" <file...>`; avoid manual staging.
- PRs should summarize scope, note testing performed, and mention any user-facing changes or new flags.
- Maintain `CHANGELOG.md`: add user-visible changes to `Unreleased` as fixes and features land, with PR/issue references and contributor thanks. Finalize that section at release; do not put later work under an already published version. The release-time-only changelog exception for `openclaw/openclaw` does not apply to this repository.
- New contributor: thank in `CHANGELOG.md` and update README contributors list if present.

## PR Workflow

- **Review mode (PR link only):** read via `gh pr view` / `gh pr diff`; do not switch branches or change code
- **Landing mode:** temp branch from `main` → bring in PR (squash default) → fix → update `CHANGELOG.md` (PR #/issue + thanks) → `make ci` → merge to `main` → delete temp branch
- **Landing mode (AGENTS.md variant, kept for the extra detail):** temp branch from `main`; bring in PR (squash default; rebase/merge when needed); fix; update `CHANGELOG.md` (PR #/issue + thanks); run `make ci`; final commit; merge to `main`; delete temp; end on `main`
- If squashing, add `Co-authored-by:` for the PR author; leave a PR comment with what landed + SHAs
- If landing contributor work, always add `Co-authored-by:` trailers for PR authors, even when we partially rewrite, group, or manually apply their changes; leave a PR comment with what landed + SHAs
- New contributor: thank in `CHANGELOG.md` and update README contributors list if present

## Security & Configuration Tips

- Never commit OAuth client credential JSON files or tokens.
- Prefer OS keychain backends; use `GOG_KEYRING_BACKEND=file` + `GOG_KEYRING_PASSWORD` only for headless environments.

## Releasing

```bash
scripts/release.sh X.Y.Z       # Tag + GitHub release + Homebrew tap update
scripts/verify-release.sh X.Y.Z
```

Always run all steps (CI + changelog + tag + GitHub release artifacts + tap update + Homebrew sanity install).
