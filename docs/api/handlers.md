# Handlers & channels

Subscribers, queryables, `get`, liveliness subscribers, matching listeners and event listeners all deliver a
stream of items. You choose **how** they're delivered.

## The options

| Handler | How you consume | When it's full |
|---|---|---|
| **Default** (`DefaultHandler`, currently a `FifoChannel` of **256**) | `recv()`, `recv_async()`, `try_recv()`, `recv_timeout()`, iteration | **Blocks** Zenoh's delivery thread until there's space |
| `FifoChannel::new(n)` | same | **Blocks** |
| `RingChannel::new(n)` | `recv()`, `try_recv()`, `recv_timeout()`, … | **Drops the oldest** item |
| `callback(f)` | `f(item)` runs on Zenoh's thread | n/a (keep `f` fast) |
| `callback_mut(f)` | as above, never called concurrently | n/a |
| Custom `IntoHandler` | your own type | your choice |

```rust
use zenoh::handlers::{FifoChannel, RingChannel};

let sub = session.declare_subscriber("a/**").with(RingChannel::new(16)).await?;
let latest = sub.try_recv()?;     // non-blocking

session.declare_subscriber("a/**")
    .callback(|sample| println!("{}", sample.key_expr()))
    .background()                 // no handle; lives until the session closes
    .await?;
```

!!! warning "A slow FIFO consumer slows everyone down"
    A full FIFO blocks the Zenoh thread that's delivering into it. That can hold up delivery to the session's
    other entities and, through back-pressure, the network. For "latest value" consumers use `RingChannel`,
    or move work off the callback quickly.

Default capacities (`zenoh/src/api/session.rs`): data reception 256, query reception 256, reply reception 256.

## Handles and lifetime

- Declaring returns an object (`Subscriber<Handler>`, `Queryable<Handler>`, …) that **derefs to the handler**,
  so `subscriber.recv_async()` works directly.
- **Dropping** the object undeclares the entity. `.undeclare()` does the same explicitly and reports errors.
- `.background()` (on subscribers, queryables, liveliness subscribers and listeners) gives up the handle:
  the entity lives until the session closes. It's only available with callbacks.

## In other languages

| Language | Callback | FIFO | Ring | Notes |
|---|---|---|---|---|
| C | `z_closure_*` | `z_fifo_channel_*_new` | `z_ring_channel_*_new` | Closures carry a context pointer and a drop function |
| C++ | lambdas | `channels::FifoChannel` | `channels::RingChannel` | |
| Python | callables | `handlers.FifoChannel` | `handlers.RingChannel` | Default is a FIFO. Callbacks run on Zenoh threads (GIL taken) |
| Kotlin | `Callback` | `ChannelHandler` (Kotlin `Channel`) | — | `Handler<T, R>` interface for custom handlers |
| Java | `Callback` | `BlockingQueueHandler` (default) | — | |
| TypeScript | callbacks | `FifoChannel` | `RingChannel` | `ChannelReceiver`: `await receive()`, `tryReceive()`, `for await` iteration |
| Go | `Closure[T]` | `NewFifoChannel[T](n)` | `NewRingChannel[T](n)` | `subscriber.Handler()` returns a Go `<-chan Sample` |
| Pico | `z_closure_*` | `z_fifo_channel_*` | `z_ring_channel_*` | Same as zenoh-c API |

## Sources

- `zenoh/src/api/handlers/` (`fifo.rs`, `ring.rs`, `callback.rs`, `mod.rs`)
- `zenoh/src/api/session.rs` (`API_*_CHANNEL_SIZE`)
