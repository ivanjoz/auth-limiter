## The crate, the config section and the systemd units all take the auth-limiter name

**Context** — This code moved into its own repository, `github.com/ivanjoz/auth-limiter`, but every
name it exposed still said `server_utils`: the crate was `genix-server-utils`, the daemon read a
`[server_utils]` TOML section, its env overrides were `SERVER_UTILS_*`, and it installed
`genix-server-utils.service`. One component answered to two names depending on which surface you
looked at.

**Decision** — Everything renamed to `auth-limiter` / `auth_limiter`: crate and binary, the TOML
section and its keys, the `AUTH_LIMITER_*` env vars, the `RUST_LOG` target, the systemd unit
triplet, and the release assets (`auth-limiter_linux_{amd64,arm64}`). Two things were deliberately
**not** renamed:

- `DOMAIN = b"genix-server-utils:v6"` in `src/service/auth.rs`. It is an HMAC domain-separation
  string mirrored byte for byte in `backend/core/auth_limiter/connection.go`. Its value is a
  protocol identifier, not a name — changing it invalidates every frame and demands a lockstep
  deploy of backend and daemon. The `:v6` suffix is the versioning channel for that; a rename is
  not a wire change and must not consume a version bump.
- The `server_utils_mem_mb` / `server_utils_cpu_percent` columns in `src/sysmetrics/writer.rs`.
  Those are live Scylla column names with rows already written against them.

**Rationale** — A single name makes the daemon greppable across four languages. The cost is a
breaking deploy: an installed host keeps running the old `genix-server-utils.service` until
`configure_auth_limiter.py` is re-run, and any `config.toml` with a `[server_utils]` section stops
being read. That was accepted deliberately for a pre-alpha project rather than carrying an alias.

## Decode the session token with the colbin crate instead of transcribing the format

**Context** — `src/bridge/token.rs` hand-wrote a colbin decoder: 577 lines mirroring the format's
`format.go`, bitstream and `typeinfo.go` to read exactly one struct, `core.UsuarioToken`. It was
pinned by vectors generated from Go, and it had already gone silently wrong. It targets
`formatVersion 0x01`, whose integer column was a frame-of-reference base plus fixed-width deltas and
whose strings were raw UTF-8; colbin v0.1.0 — which `backend/go.mod` pins — writes the `varint` array
codec and `packed5` frames, and routes every single-record message through **compact mode**. The
token the backend issues now arrives as 27 bytes whose first byte is `0x43`, bit 0 set, where this
decoder expected `0x01` and a columnar layout. Every browser SSE connection would have been rejected
with `MalformedSessionToken`, and the three test suites that pinned the old bytes were all pinning
a message the backend no longer writes.

**Decision** — Depend on `colbin`, a Rust decoder now published out of the repository that defines
the format (`github.com/ivanjoz/colbin`, `rust/`), pinned by `rev`. `token.rs` keeps only what is
genuinely this bridge's: `UserToken`, the five-field layout and the ids it implies,
`decode_session_token` over `colbin::decode_one`, `decode_session_base64`, and the channel token,
which is a separate custom format and is untouched. 577 lines down to 416, of which the channel
token and its cross-language vectors are the larger half; `TokenError`'s four colbin-internal
variants collapse into one `#[from] colbin::Error`. `auth_limiter/vectors` is a small standalone Go
module that prints the session-token vectors the tests assert, so regenerating them after a format
change is a command rather than an archaeology exercise.

**Rationale** — The failure above is the argument: the format's rules lived in one repository and a
partial transcription of them lived here, with nothing but a code review between them, and the
transcription fell three format versions behind without anything reporting it. A decoder in the repo
that defines the format sits next to the codecs it mirrors and next to a corpus generated from them,
which CI regenerates and diffs.

Pinned by `rev` rather than `tag` because that repository's tags are its Go module's version ladder:
`v0.1.0` is a commit that predates the crate, so a tag requirement would either resolve to a tag with
no crate in it or entangle two release cadences that have no reason to move together. The cost is
that updating is a deliberate act — there is no semver range to float on — which for a wire format
shared with a Go backend is the behaviour worth having.

What the crate does not carry is the standard mode's full surface: nested structs, maps and
`interface{}` columns are out, because compact mode excludes them and compact mode is what a
single-record message is. Nothing here reads an ORM blob; a Rust reader that needs one later needs
the crate widened first, which is a decision better taken when there is a caller for it.

The vectors moved with the decoder. `token.rs`, `auth.rs` and `tests/bridge_http.rs` each carried the
same stale `0x01` hex message; all three now carry the base64 the Go generator prints, and the one in
`auth.rs` still proves the Rust and Go token HMACs agree byte for byte, since its `Hash` field was
computed by `core.ComputeUsuarioTokenHash` with the same test secret.
