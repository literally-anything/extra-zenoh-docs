# Liveliness API

How liveliness works is covered in [Discovery → Liveliness](../discovery/liveliness.md). This page lists the API.

## Rust

```rust
let liveliness = session.liveliness();

// Token: alive while the token (and the session) lives
let token = liveliness.declare_token("services/db/replica-1").await?;

// Subscriber: Put = appeared, Delete = disappeared
let sub = liveliness.declare_subscriber("services/db/*")
    .history(true)                 // also get currently alive tokens
    .await?;

// Snapshot query
let replies = liveliness.get("services/db/*").timeout(Duration::from_secs(1)).await?;

token.undeclare().await?;
```

| Builder | Options |
|---|---|
| `declare_token(ke)` | — |
| `declare_subscriber(ke)` | `history(bool)` (default `false`), `callback`, `with`, `background` |
| `get(ke)` | `timeout` (default `queries_default_timeout`), `callback`, `with`, :material-flask: `cancellation_token` |

## Other languages

| Language | Token | Subscriber | Get |
|---|---|---|---|
| C | `z_liveliness_declare_token` | `z_liveliness_declare_subscriber` (`history` option) | `z_liveliness_get` |
| C++ | `Session::liveliness_declare_token` | `liveliness_declare_subscriber` | `liveliness_get` |
| Python | `session.liveliness().declare_token` | `.declare_subscriber(ke, history=…)` | `.get(ke, timeout=…)` |
| Kotlin / Java | `session.liveliness().declareToken` | `.declareSubscriber` | `.get` |
| TypeScript | `session.liveliness().declareToken` | `.declareSubscriber` | `.get` |
| Pico | `z_liveliness_declare_token` | `z_liveliness_declare_subscriber` | `z_liveliness_get` (`Z_FEATURE_LIVELINESS`, on by default) |
| Go | `session.Liveliness().DeclareToken` | `.DeclareSubscriber` | `.Get` |

## Sources

- `zenoh/src/api/liveliness.rs`, `builders/liveliness.rs`
- Binding headers and sources at 1.10.1
