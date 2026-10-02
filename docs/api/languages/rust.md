# Rust

The reference implementation. Every other binding except zenoh-pico wraps it.

## Install

```toml
[dependencies]
zenoh = "1.10.1"                                        # default features
# zenoh = { version = "1.10.1", features = ["unstable", "shared-memory"] }
zenoh-ext = { version = "1.10.1", features = ["unstable"] }   # serialization (stable), advanced pub/sub (unstable)
```

Feature flags are described on [Cargo feature flags](../../concepts/feature-flags.md). The ones that
change the **API** are:

| Feature | Adds |
|---|---|
| `unstable` | Connectivity info and events, `SourceInfo`, `Reliability`, `ReplyKeyExpr`, `TimeRange`, cancellation, `session.config()`, key-expression trees and formats, `EntityGlobalId` |
| `shared-memory` (+ `unstable`) | `zenoh::shm` |
| `internal` | APIs meant for bindings (`zenoh::internal`) |
| `plugins` | Plugin traits ([Writing a plugin](../../plugins/writing-a-plugin.md)) |

## Conventions

- **Builders everywhere.** `session.put(k, v)` returns a builder. Set options, then `.await` it (async) or
  call `.wait()` (sync, `use zenoh::Wait`).
- **Runtime**: Zenoh runs its own tokio runtimes internally. You can use any executor for `.await`, or
  none at all with `.wait()`.
- **Errors**: `zenoh::Result<T>` = `Result<T, zenoh::Error>` (a boxed error).
- **Drop = undeclare** for publishers, subscribers, queryables and tokens. Use `background()` for entities
  that should live without a handle.
- **Logging**: `zenoh::init_log_from_env_or("error")` (reads `RUST_LOG`).

## Module map

| Module | Contents |
|---|---|
| `zenoh` | `open`, `Session`, `Config`, `Wait`, `Result` |
| `zenoh::key_expr` | `KeyExpr`, `keyexpr`, `OwnedKeyExpr`; :material-flask: `format`, `keyexpr_tree` |
| `zenoh::pubsub` | `Publisher`, `Subscriber`, builders |
| `zenoh::query` | `Query`, `Queryable`, `Querier`, `Reply`, `Selector`, `Parameters`, `QueryTarget`, `ConsolidationMode` |
| `zenoh::sample` | `Sample`, `SampleKind`, `Locality`, `SampleBuilder`; :material-flask: `SourceInfo` |
| `zenoh::bytes` | `ZBytes`, `Encoding` |
| `zenoh::handlers` | `FifoChannel`, `RingChannel`, `Callback`, `DefaultHandler` |
| `zenoh::qos` | `Priority`, `CongestionControl`; :material-flask: `Reliability` |
| `zenoh::liveliness` | `Liveliness`, `LivelinessToken` |
| `zenoh::matching` | `MatchingStatus`, `MatchingListener` |
| `zenoh::scouting` | `scout`, `Hello` |
| `zenoh::session` | `SessionInfo`, `ZenohId`; :material-flask: `Transport`, `Link`, events |
| `zenoh::time` | `Timestamp`, `NTP64` |
| `zenoh::config` | `Config`, `EndPoint`, `Locator`, `WhatAmI` |
| `zenoh::shm` | :material-flask: SHM ([API](../../shm/api.md)) |
| `zenoh::cancellation` | :material-flask: `CancellationToken` |
| `zenoh_ext` | `z_serialize`/`z_deserialize`, `ZSerializer`/`ZDeserializer`; :material-flask: advanced pub/sub |

## Examples

`examples/examples/` in the main repo: `z_put`, `z_pub`, `z_sub`, `z_get`, `z_queryable`, `z_querier`,
`z_liveliness`, `z_sub_liveliness`, `z_get_liveliness`, `z_scout`, `z_info`, `z_storage`, `z_pull`,
`z_ping`/`z_pong`, `z_pub_thr`/`z_sub_thr`, SHM variants, `z_bytes`, `z_formats`. Run with
`cargo run --example z_pub -- -e tcp/127.0.0.1:7447`.

API docs: [docs.rs/zenoh](https://docs.rs/zenoh), [docs.rs/zenoh-ext](https://docs.rs/zenoh-ext).

## Sources

- `zenoh/src/lib.rs`, `zenoh/Cargo.toml`, `zenoh-ext/src/lib.rs`, `examples/`
