# CLAUDE.md

Guidance for Claude Code in this repository. Keep it short.

## What this is

A fork of `github.com/bogdanfinn/fhttp`, which is a fork of Go's `net/http` and
`golang.org/x/net/http2` that lets the caller fix the order of headers and pseudo-headers and
the HTTP/2 SETTINGS, WINDOW_UPDATE and PRIORITY frames, so a Go client can look like a browser
on the wire. The module path is still `github.com/bogdanfinn/fhttp` and must stay so. This
repository is **public**.

**Wire behaviour is byte-strict.** A change to header order, HTTP/2 frames or settings,
`accept-encoding`, or connection flow changes the fingerprint of the application that pins this
fork. It must be proven by a capture, and it is released as a new tag that the application
adopts deliberately. Read `.claude/rules/wire-behaviour.md` before you change a `.go` file.

## Ours and upstream's

- Almost everything here is upstream's. Keep it that way: do not reformat, rename or tidy
  inherited code, so that an upstream merge stays possible.
- **Ours** are two commits by this organisation, both in `http2/transport.go`:
  `ConnectionFlow = 0` sends no connection WINDOW_UPDATE, and the connection window is then
  refilled from the stream window.

## Where the releases are (checked 2026-11-12; check again before you rely on it)

- `master` is upstream's `v0.6.8` and nothing more.
- `v0.6.10`, the tag the application pins, is the tip of the branch `fix/connectionflow-zero`,
  two commits ahead of `master`. **It was never merged: `master` does not contain the
  release.** `v0.6.9` is a second tag on the same commit.
- Upstream has its own, different `v0.6.9`, and has moved on to `v0.7.x`.

So: a tag cut from `master` today would silently lose the `ConnectionFlow` behaviour. Check
with `git merge-base --is-ancestor v0.6.10 master` (exit 1 means still not merged), and follow
the release list in `.claude/rules/git-and-prs.md`.

## How a release reaches the application

The application that uses this fork pins it at a release tag, in a `replace` line of its
`go.mod`.

A release here changes nothing there. Adoption is a pull request in that repository that
changes the version on that line; it counts as a change to request bytes and owes a live A/B
test. The application also replaces `github.com/bogdanfinn/utls` with the organisation's fork,
so the `utls` version in this `go.mod` is not the one the application links: this code must
keep compiling against the fork's pinned tag.

## Commands

```bash
go build ./...          # passes
go test -vet=off -count=1 -run '^(TestHeaderOrder|TestConnectionSettings|TestWriteSettings|TestWriteSettingsAck|TestWritePriority|TestWriteWindowUpdate)$' ./http2/
                        # the offline wire tests: pass, under a second
go vet ./...            # FAILS on master (inherited test files)
go test ./...           # FAILS on master (vet, then much of the inherited suite)

git config core.hooksPath .githooks     # once per clone: turn on the commit-msg hook
powershell -NoProfile -ExecutionPolicy Bypass -File .claude/hooks/test-hooks.ps1   # the hooks' own tests
```

Never write that the tests pass. Say which command you ran (`.claude/rules/inherited-tests.md`).

## How work lands

Branch from `master`, in a git worktree; pull request into `master`; merged on green checks.
No `beta` branch. Never push to `master`, never force-push, never move a tag. No commit or pull
request credits a coding agent. Every push, pull request, merge and tag needs the owner's yes
at the time.

## Rules by area (`.claude/rules/`)

| File | Covers | The rule in one line |
|---|---|---|
| `git-and-prs.md` | every session | Pull request into `master`; how upstream is merged; the release list. |
| `choosing-a-solution.md` | every session | Pick the best solution; in a fork that is usually the smallest diff. |
| `wire-behaviour.md` | every `.go` file, `go.mod` | A wire change is proven by a capture, one per pull request and per tag. |
| `inherited-tests.md` | test files | The inherited suite is partly broken; compare failures, never claim a pass. |
