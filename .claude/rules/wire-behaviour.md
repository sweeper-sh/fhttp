---
paths:
  - "**/*.go"
  - "go.mod"
  - "go.sum"
---
# Wire behaviour is byte-strict

This package exists to put exact, browser-like bytes on the wire. The application that pins it
is fingerprinted by the servers it talks to, so a changed header order, a changed HTTP/2 frame
or a changed default is a changed fingerprint. It degrades results silently: every test here
can stay green while the wire is wrong.

## What this package decides on the wire

- **Header order, HTTP/2** (`encodeHeaders`, `http2/transport.go`): pseudo-headers in the order
  of `PHeaderOrderKey` or `Transport.PseudoHeaderOrder`; then the regular headers sorted by
  `HeaderOrderKey` (`Header.SortedKeyValuesBy`, `header.go`). A header that is not named in the
  order is written **after** the named ones, alphabetically.
- **Header order, HTTP/1.1** (`Header.writeSubset`, `header.go`): the same sort.
- **Headers it adds by itself**: `content-length` when the request has a body and does not
  name one; `accept-encoding: gzip, deflate, br` when the request names none and compression
  is on; `user-agent: Go-http-client/2.0` when the request names none; `cookie` split into one
  field per crumb on HTTP/2.
- **The HTTP/2 connection opening** (`newClientConn`, `http2/transport.go`): the preface, one
  SETTINGS frame built from `Transport.Settings` in `SettingsOrder`, a WINDOW_UPDATE from
  `ConnectionFlow`, the PRIORITY frames in `Priorities`, the first stream ID
  (`InitialStreamID`) and the priority on HEADERS (`HeaderPriority`).
- **Flow control after that**: when and by how much WINDOW_UPDATE is sent
  (`transportResponseBody.Read`), and the HPACK table size (`HeaderTableSize`).

The callers build an `http2.Transport` from package `http2` directly. `h2_bundle.go` in the root
package is an older bundled copy without `SettingsOrder` or `ConnectionFlow`; a change in
`http2/` is not mirrored there, and nothing here assumes it is.

## The rules

1. **Any change to the list above is a wire change, whatever the reason for it**: a bug fix, a
   refactor, a merge from upstream, a dependency bump that touches `golang.org/x/net` or HPACK.
   Treat "it only reorders an internal map" as a wire change until a capture says otherwise.
2. **A wire change is proven by a capture**, before and after, of the bytes on the wire: the
   HTTP/2 frames in order with their values, and the header list in order. Put the comparison
   in the pull request. A passing test here is not that proof.
3. **A default never changes silently.** If a zero value starts to mean something else (as
   `ConnectionFlow = 0` did between `v0.6.8` and `v0.6.10`), the pull request says so in its
   first line and the release notes it, because every caller that relied on the old meaning
   changes behaviour without changing its own code.
4. **One wire change per pull request and per release tag**, so the application can adopt it,
   test it against live traffic and revert it alone.
5. **The module path stays `github.com/bogdanfinn/fhttp`.** The application reaches this fork
   through a `replace` directive, which only works while the path in `go.mod` is unchanged.
6. **Exported names the callers use do not change shape**: the `http2.Transport` fields above,
   `HeaderOrderKey`, `PHeaderOrderKey`, `http2.Setting`, `http2.Priority`, `PriorityParam`.
7. **This repository is public.** No capture file, no hostname of a site the application
   targets, and nothing about the application's internals goes into code, a comment, a commit
   message or a pull request here. Describe the wire fact ("Chrome 152 sends no connection
   WINDOW_UPDATE"), not who needs it.

## Changing wire behaviour

- [ ] You can state, in one sentence, which bytes change and for which callers.
- [ ] Before and after captures are compared and the comparison is in the pull request.
- [ ] A test pins the new behaviour at the frame or header level, offline (no network), in
      `http2/`: see `TestHeaderOrder` and `TestConnectionSettings` for the style.
- [ ] `go build ./...` and the offline wire tests pass (`inherited-tests.md` has the command).
- [ ] The pull request names what was **not** verified.

## What enforces this today

- `TestHeaderOrder` (`http2/header_order_test.go`), `TestConnectionSettings`
  (`http2/fhttp_test.go`), `TestWriteSettings`, `TestWritePriority`, `TestWriteWindowUpdate`
  (`http2/frame_test.go`) and `TestHTTP1HeaderOrder` (`header_test.go`) pin a little of it.
- **Nothing pins the connection opening as a whole**: no test fixes the order of SETTINGS, the
  WINDOW_UPDATE value or its absence, the PRIORITY frames or the added headers. The
  `ConnectionFlow = 0` behaviour of `v0.6.10` has no test.
- No check on a pull request compares captures. Rules 2 to 4 and 7 ride on discipline.
- The real gate is downstream: the application adopts a tag by a pull request of its own that
  owes a live A/B test before its release.
