# Liveliness

Liveliness gives you **application-level presence**: "is robot 7 alive?", "which workers are online?". It
works by declaring **tokens** on key expressions. A token exists exactly as long as the session that
declared it, so a crash or network partition makes it disappear.

## The three operations

| Operation | API (Rust) | Result |
|---|---|---|
| Declare a token | `session.liveliness().declare_token("robots/7")` | Token lives until it's undeclared or dropped, or until the session or its connectivity is lost |
| Subscribe to changes | `session.liveliness().declare_subscriber("robots/*")` | `Put` sample when a matching token appears, `Delete` when it disappears |
| Query current tokens | `session.liveliness().get("robots/*")` | One reply per live matching token |

Every binding has these three (see the [API matrix](../api/index.md)).

### Subscriber `history`

```rust
session.liveliness().declare_subscriber("robots/*").history(true)
```

- `history(false)` (default): the subscriber doesn't query the network for tokens that already exist.
  Already-live tokens may still be delivered, depending on what the routing layer already knows.
- `history(true)`: when declared, the subscriber also queries for currently live tokens (the same set a
  liveliness `get` would return), so you start with a full picture.

### `get` timeout

`liveliness().get()` uses `queries_default_timeout` (10 s) unless you call `.timeout(...)`.

## When a token disappears

A liveliness subscriber gets a `Delete` when:

- the token is undeclared, or dropped (with its session);
- the declaring session closes;
- the **transport** to the declaring node (or to the router path towards it) is lost, which is detected by
  the [lease](../transports/transport-layer.md#leases-and-keep-alives) (10 s by default). Shorten
  `transport/link/tx/lease` for faster detection.

## Typical uses

- Service discovery: each worker declares `services/<name>/<instance>`, and clients list them with `get`
  and watch them with a subscriber.
- Failure detection: a supervisor watches `robots/*` and reacts to `Delete`.
- [Advanced pub/sub](../api/advanced-pubsub.md) uses liveliness tokens under `@adv/...` to find publishers
  and their caches.

## Access control

ACL rules can allow or deny `liveliness_token`, `declare_liveliness_subscriber` and `liveliness_query`
separately. See [Access control](../security/access-control.md).

## Admin space

Tokens known to a router are listed under `@/<zid>/router/token/**`.

## Sources

- `zenoh/src/api/liveliness.rs`, `zenoh/src/api/builders/liveliness.rs`
- `examples/examples/z_liveliness.rs`, `z_sub_liveliness.rs`, `z_get_liveliness.rs`
