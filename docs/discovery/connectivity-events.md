# Connectivity events

Zenoh 1.10 can tell a session which **transports** (remote nodes) and **links** (connections) it has, and
notify it when they change. :material-flask: These APIs need the `unstable` feature.

## Snapshot: `session.info()`

| Call | Returns | Stability |
|---|---|---|
| `info().zid()` | This session's ZID | stable |
| `info().routers_zid()` | ZIDs of connected routers | stable |
| `info().peers_zid()` | ZIDs of connected peers | stable |
| `info().locators()` | This session's listening locators (without loopback) | :material-flask: unstable |
| `info().transports()` | `Transport` for each remote node | :material-flask: unstable |
| `info().links()` (optionally `.transport(t)`) | `Link` for each connection | :material-flask: unstable |

### `Transport` fields

| Method | Meaning |
|---|---|
| `zid()` | Remote ZID |
| `whatami()` | Remote mode |
| `is_qos()` | QoS negotiated on this transport |
| `is_shm()` | Shared memory negotiated |
| `is_multicast()` | Multicast transport |

### `Link` fields

| Method | Meaning |
|---|---|
| `zid()` | Remote ZID |
| `src()`, `dst()` | Local and remote locators |
| `group()` | Multicast group locator, if any |
| `mtu()` | Link MTU |
| `is_streamed()` | Byte-stream link (TCP, TLS, …) |
| `interfaces()` | Local interface names used by the link |
| `auth_identifier()` | Authenticated identity (for example the TLS certificate CN) |
| `priorities()` | Priority range (`prio` metadata) |
| `reliability()` | Reliability (`rel` metadata or the link default) |

## Events

```rust
use zenoh::sample::SampleKind;

let listener = session.info().transport_events_listener().history(true).await?;
while let Ok(event) = listener.recv_async().await {
    match event.kind() {
        SampleKind::Put => println!("connected to {}", event.transport().zid()),
        SampleKind::Delete => println!("lost {}", event.transport().zid()),
    }
}
```

- `transport_events_listener()` yields `TransportEvent { kind, transport }`. `Put` means opened and `Delete` means closed.
- `link_events_listener()` yields `LinkEvent { kind, link }`, and can be limited to one transport with `.transport(t)`.
- `.history(true)` first replays a `Put` for every transport or link that already exists.
- Like other listeners, they take `callback`, `with(handler)` and `background()`.

## Without the unstable API: the session-local admin space

Every session also publishes its own transport and link events as ordinary samples on
`@/<own zid>/session/transport/**` (visible only inside that session). A normal subscriber works:

```rust
let me = session.zid();
let sub = session.declare_subscriber(format!("@/{me}/session/transport/**")).await?;
// Put  @/<me>/session/transport/unicast/<peer>             {"zid":…,"whatami":…,"is_qos":…,"is_shm":…}
// Put  @/<me>/session/transport/unicast/<peer>/link/<hash>  {"src":…,"dst":…,"mtu":…,…}
// Delete on close
```

See the [admin space reference](../admin-space/reference.md#session-local-admin-space-zidsession).

## When to use what

| Need | Use |
|---|---|
| Know when **a specific application** is up or down | [Liveliness](liveliness.md) tokens (survive routing through intermediate routers) |
| Know whether **anyone** subscribes to my data | [Matching](matching.md) |
| Monitor **network-level** connectivity of this process | Transport/link events |
| Monitor **any** node from outside | The [admin space](../admin-space/index.md) (`@/<zid>/<mode>` sessions list) |

## Sources

- `zenoh/src/api/info.rs`, `connectivity.rs`
- `zenoh/src/api/builders/info_transport.rs`, `info_links.rs`
- `zenoh/src/lib.rs` (`pub mod session`, unstable re-exports)
