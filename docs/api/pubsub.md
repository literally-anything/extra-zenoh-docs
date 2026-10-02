# Publish / subscribe

## One-shot put and delete

```rust
session.put("robot/arm/angle", "42").await?;
session.put("robot/arm/angle", payload)
    .encoding(Encoding::APPLICATION_JSON)
    .priority(Priority::RealTime)
    .congestion_control(CongestionControl::Block)
    .express(true)
    .attachment("trace-id=abc")
    .timestamp(session.new_timestamp())
    .await?;
session.delete("robot/arm/angle").await?;
```

## Publisher

A publisher is a declared, reusable sender on one key expression. It declares the key expression once,
can report [matching](../discovery/matching.md) status, and its QoS applies to every message it sends.

```rust
let publisher = session.declare_publisher("robot/camera")
    .encoding(Encoding::IMAGE_JPEG)
    .priority(Priority::DataHigh)
    .congestion_control(CongestionControl::Drop)
    .express(false)
    // .reliability(Reliability::BestEffort)       // unstable
    // .allowed_destination(Locality::Remote)      // unstable
    .await?;

publisher.put(frame).await?;
publisher.delete().await?;
let listener = publisher.matching_listener().await?;
```

### Publication options

| Option | Default | Meaning |
|---|---|---|
| `encoding` | `zenoh/bytes` | Payload encoding |
| `priority` | `Data` | Queue priority |
| `congestion_control` | `Drop` | Drop or block when the queue is full ([details](../transports/transport-layer.md#congestion-control)) |
| `express` | `false` | Skip batching |
| `reliability` :material-flask: | `Reliable` | Link-selection hint |
| `allowed_destination` :material-flask: | `Any` | Deliver only to `SessionLocal` or `Remote` subscribers |
| `attachment` (per put) | none | Extra bytes, delivered as `Sample::attachment()` |
| `timestamp` (per put) | none | Explicit timestamp |
| `source_info` (per put) :material-flask: | none | Source entity and sequence number |

Configuration can **override** some of these per key expression without code changes:
[`qos/publication`](../configuration/qos-overwrite.md#qospublication).

## Subscriber

```rust
let subscriber = session.declare_subscriber("robot/**").await?;
while let Ok(sample) = subscriber.recv_async().await {
    println!("{} {:?}: {}", sample.key_expr(), sample.kind(), sample.payload().try_to_string()?);
}
```

| Option | Meaning |
|---|---|
| `callback(f)` / `callback_mut(f)` | Run a callback for each sample instead of using a channel ([Handlers](handlers.md)) |
| `with(handler)` | Use a `FifoChannel`, `RingChannel` or custom handler |
| `background()` | Keep the subscriber alive until the session closes without holding a handle |
| `allowed_origin(Locality)` :material-flask: | Only receive local or remote publications |

A subscriber receives both `Put` and `Delete` samples. Check `sample.kind()`.

## Pull-style consumption

There's no separate "pull subscriber" in 1.x. Use a **ring channel** (`RingChannel::new(n)`) to keep the
latest *n* samples, and read them when you want with `try_recv()`. `examples/z_pull.rs` shows this.

## Delivery semantics

- **Ordering**: samples from one publisher on one priority arrive in order.
- **Reliability**: with reliable links (TCP/TLS/QUIC), messages aren't lost on the link, but they **can be
  dropped** under congestion with `CongestionControl::Drop`, by [downsampling](../configuration/downsampling.md),
  [low-pass filters](../configuration/low-pass-filter.md) or [ACL](../security/access-control.md). For
  end-to-end loss detection and recovery, use [advanced pub/sub](advanced-pubsub.md).
- **Late joiners**: a subscriber doesn't get data published before it existed. Use a storage
  (`get` on start), or advanced pub/sub `history`.
- **Local delivery**: a session's own subscribers receive its own publications (unless `allowed_destination`
  says otherwise), without going over the network.

## Sources

- `zenoh/src/api/publisher.rs`, `subscriber.rs`, `builders/publisher.rs`, `builders/subscriber.rs`
- `examples/examples/z_pub.rs`, `z_sub.rs`, `z_pull.rs`, `z_put.rs`, `z_delete.rs`
