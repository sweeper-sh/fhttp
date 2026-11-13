---
paths:
  - "**/*_test.go"
  - "**/testdata/**"
---
# The inherited test suite

Most tests here came with the fork of `net/http` and `golang.org/x/net/http2`. Many no longer
pass, for reasons that have nothing to do with a change you make: the fork changed header
order, defaults and TLS, and the suite was never brought along. **Never write "the tests pass"
about this repository.** Say exactly which command you ran and what it printed.

## What happens today (measured on `master`, Go 1.26.1, Windows, 2026-11-12)

- `go build ./...` passes.
- `go vet ./...` **fails**: 45 findings, all in test files (non-constant format strings,
  unkeyed fields; `cgi/child_test.go` does not type-check).
- `go test ./...` **fails before running** the root package, `cgi` and `http2`, because `go
  test` runs vet first. `httptest` and `httputil` run and fail.
- `go test -vet=off ./...`: the root package and `http2` were still running when stopped at
  400 seconds and had reported 75 failed tests by then; `cgi` does not compile. `cookiejar`,
  `fcgi`, `http2/h2c`, `http2/hpack`, `httptrace`, `internal`, `internal/profile` and `pprof`
  pass.
- Some tests need the network (`TestHeaderOrder2` calls httpbin.org, `TestHeaderOrder3` calls
  google.com and fails). A root-package run can also fail after its tests passed, on the
  goroutine-leak check in `main_test.go`.

Other Go versions and Linux were not measured. Do not assume they are better.

## The tests that mean something for the wire, and run offline

```bash
go test -vet=off -count=1 -run '^(TestHeaderOrder|TestConnectionSettings|TestWriteSettings|TestWriteSettingsAck|TestWritePriority|TestWriteWindowUpdate)$' ./http2/
```

This passes on `master` in under a second. It is what the stop hook runs. It is a floor, not a
proof (`wire-behaviour.md`).

## The rules

1. **Compare, do not conclude.** To know whether you broke something, run the same command on
   your base commit and on your change and compare the lists of failing tests. A test that
   failed before and fails after is not yours; a new name in the list is.
2. **Do not delete, skip or loosen an inherited test to get green.** A broken test is left as
   it is, or repaired for real as its own change.
3. **A test you add runs offline and alone**: no outside host, no fixed port, no dependence on
   another test. Put wire tests in `http2/`, where a narrow `-run` works; the root package's
   `TestMain` leak check can fail a narrow run.
4. **Do not "fix" vet findings in bulk inside a wire change.** Cleaning the suite is welcome,
   as separate pull requests that touch only `_test.go` files.
5. **Say what was not run.** "Offline wire tests pass; the rest of the suite was not run
   because it fails on `master`" is a true sentence. "Tests pass" is not.

## What enforces this today

- `.claude/hooks/verify-before-stop.ps1` (agents only) runs `go build ./...` and the command
  above when a Go file changed.
- No workflow runs on a pull request as this is written, and no list of known failures is
  kept, so rule 1 is done by hand each time.
