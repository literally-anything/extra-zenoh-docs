# Cancellation

A `CancellationToken` stops in-flight **get** operations early: session `get`, querier `get` and liveliness
`get`. :material-flask: Unstable in Rust/C/C++/Python; available in TypeScript and Go.

```rust
use zenoh::cancellation::CancellationToken;

let token = CancellationToken::default();
let replies = session.get("slow/**")
    .callback(|reply| { /* … */ })
    .cancellation_token(token.clone())
    .await?;

// later, from anywhere:
token.cancel().await?;    // after this returns, the callback is never called again
```

Semantics (`zenoh/src/api/cancellation.rs`):

- `cancel()` interrupts **every** get associated with the token. If a callback is running, `cancel()` waits
  for it to finish. When it returns, no callback will run again.
- Gets started with an already-cancelled token are cancelled straight away.
- `is_cancelled()` reports the state.
- One token can be shared by many gets, for example to cancel everything a UI view started.

| Language | Type |
|---|---|
| C | `z_cancellation_token_new` / `_cancel` / `_is_cancelled` (unstable), passed in the get options |
| C++ | `zenoh::CancellationToken` (unstable) |
| Python | `zenoh.CancellationToken` (unstable) |
| TypeScript | `CancellationToken` (`cancellationToken` option of `get`) |
| Go | `zenoh.NewCancellationToken()` |
| Pico | `z_cancellation_token_*` (unstable) |
| Kotlin / Java | ❌ |

## Sources

- `zenoh/src/api/cancellation.rs`, `builders/query.rs`, `builders/querier.rs`, `builders/liveliness.rs`
