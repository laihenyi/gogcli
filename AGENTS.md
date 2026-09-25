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

Tracking worker: `pnpm -C internal/tracking/worker install --frozen-lockfile`, then `make worker-ci`.

Remote checks: after `crabbox warmup --keep --timing-json`, hydrate this repository with `crabbox actions hydrate --id <id> --github-runner`; its workflow uses Actions cache semantics that the local adapter does not support. Run `crabbox run --id <id> --timing-json --shell -- "make ci"`, reuse the lease across checks, and stop it when finished with `crabbox stop <id>`.

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

- Formatting: `make fmt` (goimports local prefix `github.com/openclaw/gogcli`, matching `go.mod` and the Makefile, + gofumpt)
- Gmail labels: IDs are case-sensitive opaque tokens; only case-fold names for lookup
- Commits: Conventional Commits with action-oriented subjects (e.g. `feat(cli): add --verbose to send`)
- Group related changes; avoid bundling unrelated refactors
- Unit tests: stdlib `testing` + `httptest` for HTTP mocking; test files next to source
- Headless keyring: set `GOG_KEYRING_BACKEND=file` + `GOG_KEYRING_PASSWORD` for CI/serverless; never commit OAuth credential JSON files or tokens

## Commit & Pull Request Guidelines

- Create commits with `committer "<msg>" <file...>` when that helper is installed; otherwise `git commit` the listed files directly. Avoid staging unrelated files.
- PRs should summarize scope, note testing performed, and mention any user-facing changes or new flags.
- Maintain `CHANGELOG.md` in every PR: add user-visible changes to `Unreleased` as fixes and features land, with PR/issue references and contributor thanks. Finalize that section at release; do not put later work under an already published version.
- New contributor: thank in `CHANGELOG.md` and update README contributors list if present.

## PR Workflow

- **Review mode (PR link only):** read via `gh pr view` / `gh pr diff`; do not switch branches or change code
- **Landing mode:** temp branch from `main`; bring in the PR (squash by default; rebase or merge when needed); fix; update `CHANGELOG.md` (PR #/issue + thanks); run `make ci`; final commit; merge to `main`; delete the temp branch; end on `main`
- Always add `Co-authored-by:` trailers for PR authors when landing contributor work, even when the changes are partially rewritten, grouped, or applied by hand; leave a PR comment with what landed + SHAs

## Security & Configuration Tips

- Never commit OAuth client credential JSON files or tokens.
- Prefer OS keychain backends; use `GOG_KEYRING_BACKEND=file` + `GOG_KEYRING_PASSWORD` only for headless environments.

## Releasing

```bash
scripts/release.sh X.Y.Z       # Tag + GitHub release + Homebrew tap update
scripts/verify-release.sh X.Y.Z
```

Always run all steps (CI + changelog + tag + GitHub release artifacts + tap update + Homebrew sanity install).

## Cross-Harness Handoff

`.wolf/` holds session handoff state shared across agents (Claude Code, Pi, Codex). Read `.wolf/OPENWOLF.md` at session start; if `.wolf/handoff.md` exists, read it fully before doing anything else. Write a new handoff with `/handoff` before switching sessions or when open items remain. `.wolf/handoff.md` is gitignored; `.wolf/OPENWOLF.md` is committed.
