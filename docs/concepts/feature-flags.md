# Cargo feature flags

Much of Zenoh is compiled in or left out by Cargo features of the `zenoh` crate. Language bindings and
`zenohd` choose their own sets, so a config key can be valid and still do nothing because the feature
behind it wasn't compiled in.

## Default features

From `zenoh/Cargo.toml`:

```toml
default = [
  "auth_pubkey", "auth_usrpwd",
  "transport_compression", "transport_multilink",
  "transport_quic", "transport_quic_datagram",
  "transport_tcp", "transport_tls", "transport_udp",
  "transport_unixsock-stream", "transport_ws",
]
```

## All features

| Feature | Default | What it enables | Related docs |
|---|---|---|---|
| `auth_pubkey` | ✅ | RSA public-key authentication (`transport/auth/pubkey`) | [Authentication](../security/authentication.md#public-key) |
| `auth_usrpwd` | ✅ | User/password authentication (`transport/auth/usrpwd`) | [Authentication](../security/authentication.md#userpassword) |
| `transport_compression` | ✅ | On-the-fly batch compression (`transport/unicast/compression`, `transport/multicast/compression`) | [Transport layer](../transports/transport-layer.md#compression) |
| `transport_multilink` | ✅ | Several links per unicast transport (`transport/unicast/max_links` > 1) | [Transport layer](../transports/transport-layer.md#multilink) |
| `transport_tcp` | ✅ | `tcp/` locators. Also sets the default listeners (`tcp/[::]:7447` for routers, `tcp/[::]:0` for peers) | [TCP](../transports/tcp.md) |
| `transport_udp` | ✅ | `udp/` locators, unicast and multicast | [UDP](../transports/udp.md) |
| `transport_tls` | ✅ | `tls/` locators | [TLS](../transports/tls.md) |
| `transport_quic` | ✅ | `quic/` locators (reliable streams) | [QUIC](../transports/quic.md) |
| `transport_quic_datagram` | ✅ | `quic/...?rel=0` best-effort datagrams | [QUIC datagram](../transports/quic-datagram.md) |
| `transport_ws` | ✅ | `ws/` WebSocket locators | [WebSocket](../transports/ws.md) |
| `transport_unixsock-stream` | ✅ | `unixsock-stream/` Unix domain sockets | [Unix socket](../transports/unixsock-stream.md) |
| `transport_unixpipe` | ❌ | `unixpipe/` named pipes | [Unix pipe](../transports/unixpipe.md) |
| `transport_serial` | ❌ | `serial/` UART links | [Serial](../transports/serial.md) |
| `transport_vsock` | ❌ | `vsock/` VM sockets (Linux) | [VSOCK](../transports/vsock.md) |
| `shared-memory` | ❌ | SHM transport. With `unstable` as well, the `zenoh::shm` API. The `transport/shared_memory` config does nothing without it | [Shared memory](../shm/index.md) |
| `uring` | ❌ | io_uring RX path for unicast links (Linux on x86_64, aarch64, riscv64, loongarch64, powerpc64) | [io_uring](../transports/io-uring.md) |
| `stats` | ❌ | Prometheus/OpenMetrics statistics in the admin space (`@/<zid>/<mode>/metrics`) and per-key stats (`stats` config) | [Statistics](../configuration/stats.md) |
| `unstable` | ❌ | APIs that may change: connectivity info and events (`session.info().transports()`/`links()`, event listeners), `SourceInfo`, `Reliability`, `TimeRange`, `ZenohParameters`, entity IDs, key-expression trees and formats, `block_first`, `allowed_destination`, the `zenoh::shm` API, cancellation tokens, timestamp stack, the `Notifier` config handle, TOML config files | — |
| `internal` | ❌ | Internal APIs used by the language bindings. Not for applications | — |
| `internal_config` | ❌ | Declared but not referenced anywhere else in the 1.10.1 workspace | — |
| `plugins` | ❌ | Plugin APIs for `zenohd` (internal and unstable) | [Plugins](../plugins/index.md) |
| `runtime_plugins` | ❌ | Dynamic plugin loading. Implies `plugins` | [Plugins](../plugins/index.md) |
| `tracing-instrument` | ❌ | Developer feature: tracing spans on async tasks | — |
| `test` | ❌ | Test helpers | — |

!!! note "`uring` is missing from the crate docs"
    The feature list in `zenoh/src/lib.rs` (docs.rs) doesn't mention `uring`. It is declared in
    `zenoh/Cargo.toml` as `uring = ["zenoh-transport/uring"]`.

## What `zenohd` is built with

`zenohd/Cargo.toml` uses `default-features = false` and turns on:

```toml
features = ["internal", "plugins", "runtime_plugins", "unstable"]
```

plus its own `default = ["zenoh/default"]`. It has an optional `shared-memory` feature that forwards to
`zenoh/shared-memory`. So a stock `zenohd`:

- has the default transports, authentication and compression;
- has the unstable APIs and dynamic plugin loading;
- does **not** have shared memory, stats, io_uring, serial, unixpipe or vsock unless you rebuild it, for
  example `cargo build -p zenohd --features zenoh/stats,zenoh/transport_serial,shared-memory`.

## Checking what was compiled in

`zenoh::FEATURES` is a string constant listing the enabled features (for example
`" zenoh/auth_pubkey zenoh/auth_usrpwd ..."`). `zenohd`'s test suite uses it to check the default set.

## Sources

- `zenoh/Cargo.toml` (`[features]`)
- `zenoh/src/lib.rs` (crate docs, `FEATURES`)
- `zenohd/Cargo.toml`, `zenohd/src/main.rs` (`test_default_features`)
