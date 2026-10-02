# Namespace, aggregation, timestamping

These options change how a session presents itself to the network: the keys it uses, how many
declarations it sends, and how its data is timestamped.

## Namespace

```json5
namespace: "site-a/robot-7",
```

When `namespace` is set, the session:

- adds the namespace as a prefix to every key expression it **sends** (puts, deletes, queries, replies,
  declarations, liveliness tokens), so `session.put("arm/angle", ...)` goes out as `site-a/robot-7/arm/angle`;
- strips the prefix from every key expression it **receives**, so a subscriber on `arm/**` sees `arm/angle`.

Rules:

- The value must be a valid key expression **without wildcards** (`OwnedNonWildKeyExpr`).
- It applies to this session only, not to anything the session routes for other nodes.
- Messages whose key doesn't start with the namespace never reach the session's entities.

Use it to run several copies of the same application on one network without changing any code.

## Aggregation

```json5
aggregation: {
  subscribers: ["robot/sensors/**"],
  publishers:  ["robot/actuators/**"],
},
```

Normally each subscriber or publisher is announced to the network separately. With aggregation, when a
local subscriber's key expression is **included** in one of the listed expressions:

1. The first such subscriber is announced to the network as a subscriber on the **aggregate** expression
   (`robot/sensors/**`), not on its own key.
2. Later subscribers covered by the same aggregate don't announce anything; they share the first one's
   declaration.

Publishers work the same way with `aggregation/publishers` (this affects matching and publisher-side
declarations).

Pros and cons:

- ✅ Far fewer declarations, and smaller routing tables in routers, for an application with many
  fine-grained subscribers.
- ⚠️ The network sees interest in the **whole** aggregate. Any data matching `robot/sensors/**` is sent to
  this session, even keys no local subscriber wants. Those are filtered locally.

Even without aggregation, two local declarations with **identical** key expressions share one network
declaration.

## Timestamping

```json5
timestamping: {
  enabled: { router: true, peer: false, client: false },
  drop_future_timestamp: false,
},
```

When `enabled` is true for the node's mode, the node creates a **Hybrid Logical Clock** (HLC, the
[`uhlc`](https://crates.io/crates/uhlc) crate) whose ID is the node's ZID. For every **put** it routes:

- **No timestamp**: it adds one from its HLC.
- **Has a timestamp**: it updates its HLC with that timestamp. If the timestamp is further ahead of local
  time than the HLC's maximum drift:
    - `drop_future_timestamp: false` (default): the timestamp is **replaced** with a fresh local one, and an
      error is logged.
    - `drop_future_timestamp: true`: the message is **dropped**, and an error is logged.

Deletes aren't timestamped by this mechanism.

The maximum drift is the `uhlc` default of **500 ms**. It can be changed with the environment variable
`UHLC_MAX_DELTA_MS` (read by `uhlc`, not by Zenoh's config).

Why it matters:

- [Storages](../plugins/storage-manager.md) need timestamps to order updates and resolve conflicts.
  Replicated storages **require** them.
- [Advanced subscribers](../api/advanced-pubsub.md) and `Sample::timestamp()` rely on them.
- Peers don't timestamp by default. Data that only passes between peers has no timestamp unless the
  publisher sets one (for example `put(...).timestamp(session.new_timestamp())`) or you turn on
  `timestamping/enabled` for peers.

`zenohd --no-timestamp` turns it off for every mode.

## Query timeout

```json5
queries_default_timeout: 10000,   // ms
```

The timeout used by `session.get()` and queriers when the call doesn't set one. When it expires, the reply
channel is closed and replies that arrive later are discarded.

## Sources

- `zenoh/src/net/routing/namespace.rs`, `zenoh/src/net/runtime/mod.rs`
- `zenoh/src/api/session.rs` (aggregation in `declare_subscriber_inner` / `declare_publisher_inner`)
- `zenoh/src/net/routing/dispatcher/pubsub.rs` (`treat_timestamp!`), `dispatcher/tables.rs`
- `uhlc` 0.8.2, the version in zenoh's `Cargo.lock` (`DEFAULT_DELTA_MS = 500`, `UHLC_MAX_DELTA_MS`)
