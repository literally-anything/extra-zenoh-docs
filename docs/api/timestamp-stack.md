# Timestamp stack (latency instrumentation)

:material-flask: Unstable, new in 1.10. Rust (`zenoh::timestamp_stack`) and Python only.

The **timestamp stack** records a timestamp at each point a message passes through: when the application
sends it, at **every routing step**, and when the destination application receives it. The receiver gets the
full list, so it can see where the latency was spent, hop by hop, without any external tracing system.

## Using it

```rust
use zenoh::timestamp_stack::{InterceptionPoint, TimestampInstrumentationBuilder};

let instr = TimestampInstrumentationBuilder::new()
    .set_send(true)
    .set_route(true)
    .set_receive(true)
    .build()?;                       // error if no point is enabled

publisher.put("data").timestamp_instrumentation(instr).await?;
// also on session.put/delete, session.get, querier.get, and query.reply*

let sample = subscriber.recv_async().await?;
if let Some(stack) = sample.timestamp_stack() {
    for r in stack.records() {
        println!("{:?} {:?} custom={}", r.point(), r.timestamp(), r.is_custom());
    }
}
```

The stack is available on `Sample`, `Query` and `ReplyError` (`timestamp_stack()`). Messages sent without
instrumentation carry no stack (`None`) and cost nothing extra.

| Point | Where it's recorded |
|---|---|
| `Send` | In the sending session (put, delete, get, reply) |
| `Route` | In the routing layer of **every node that forwards the message**, including the sender's and receiver's own runtimes |
| `Receive` | In the receiving session, just before your callback runs |

A peer-to-peer `put` with all three points enabled arrives with four records:
**Send → Route (sender's runtime) → Route (receiver's runtime) → Receive**. Going through one router adds a
third `Route`. (Checked with 1.10.1.)

## Timestamp format

By default each record holds a **UHLC timestamp** from the node's clock, the same hybrid logical clock
used for sample timestamps. If the node has no HLC (`timestamping` disabled), one is created on demand for
this feature. Clocks aren't synchronised between machines, so cross-host differences are only as good as
your clock sync (NTP/PTP).

You can supply your own format instead:

```rust
let session = zenoh::open(config)
    .with_timestamp_callback(|ctx| {
        // ctx.zid, ctx.whatami
        my_monotonic_ns().to_le_bytes().to_vec()
    })
    .await?;
```

Records from that node then have `is_custom() == true` and hold `InstrumentationTimestamp::Custom(bytes)`.
If the callback returns an empty `Vec`, no record is written. Python has the same option:
`zenoh.open(config, timestamp_callback=…)`.

## Limits

- At most **255 records** per message (`MAX_STACK_SIZE`). Points past that are silently skipped.
- Each node records only if its own build has the `unstable` feature. A stock `zenohd` has it. Nodes
  without it forward the stack unchanged but add no `Route` record.
- The stack travels as the `TsStack` extension (`0x7`) on PUSH, REQUEST and RESPONSE. See
  [Wire protocol](../architecture/protocol.md#extensions).

## Sources

- `zenoh/src/api/timestamp_stack.rs`, `zenoh/src/api/builders/session.rs` (`with_timestamp_callback`)
- `zenoh/src/net/routing/dispatcher/pubsub.rs`, `queries.rs` (`Route` records)
- `commons/zenoh-protocol/src/network/timestamp_stack.rs` (`MAX_STACK_SIZE`, flags)
- `zenoh-python/zenoh/__init__.pyi`
