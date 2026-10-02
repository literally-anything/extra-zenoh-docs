# zenoh-ext: Group, PublicationCache, QueryingSubscriber

Besides serialization and [advanced pub/sub](advanced-pubsub.md), the Rust `zenoh-ext` crate has a few
APIs that aren't documented on zenoh.io. They all need `zenoh-ext`'s **`unstable`** feature.

| API | Status at 1.10.1 | Replacement |
|---|---|---|
| `group::Group` (membership) | Unstable, **not** deprecated, no bindings | — (liveliness tokens cover most uses) |
| `PublicationCache` (`SessionExt::declare_publication_cache`) | **Deprecated** | `AdvancedPublisher` with `.cache(...)` |
| `QueryingSubscriber` / `FetchingSubscriber` (`.querying()`, `.fetching()`) | **Deprecated** | `AdvancedSubscriber` with `.history(...)` |
| `SubscriberForward` (`sub.forward(sink)`) | Unstable | — |

The deprecated APIs are still worth knowing: older applications use them, the bindings still expose them
(zenoh-c `ze_declare_querying_subscriber` / `ze_declare_publication_cache` and their `background`
variants, zenoh-cpp `ext::SessionExt::declare_querying_subscriber` / `declare_publication_cache`), and they
work with storages.

## Group membership (`zenoh_ext::group`)

A small group-membership protocol built on pub/sub and queries:

```rust
use std::{sync::Arc, time::Duration};
use zenoh_ext::group::*;

let z = Arc::new(zenoh::open(zenoh::Config::default()).await?);
let me = Member::new(z.zid().to_string())?          // member ID: key expression without wildcards
    .info("worker in rack 3")                          // optional free-form string
    .lease(Duration::from_secs(3));                    // default 18 s
let group = Group::join(z.clone(), "zgroup", me).await?;  // group ID: no wildcards

let events = group.subscribe().await;   // flume::Receiver<GroupEvent>
group.view().await;                      // Vec<Member>, including yourself
group.size().await;
group.wait_for_view_size(3, Duration::from_secs(15)).await;  // -> bool
group.leader().await;                    // the member with the greatest ID
```

### Wire behaviour

| Key expression | Use |
|---|---|
| `zenoh/ext/net/group/<group>/evt` | Each member publishes `Join` (once, at join) and `KeepAlive` events here, and subscribes to it |
| `zenoh/ext/net/group/<group>/<member>` | Each member's queryable. Replies with its `Member` record |

- Events and member records are **bincode**-encoded Rust structs, so only Rust programs using the same
  `zenoh-ext` version can join the same group.
- Keep-alives are sent every `lease × refresh_ratio` (default 18 s × 0.75 = 13.5 s), at priority
  `DataHigh` by default (`.priority()`).
- A watchdog runs every second and drops members whose lease has run out, raising `LeaseExpired`.
- When a keep-alive comes from an unknown member, the receiver queries that member's key, adds it to its
  view and raises `Join`.

### Quirks (from the source)

- **`Leave` is never sent.** Dropping a `Group` stops its tasks but publishes nothing, so other members only
  notice after the lease runs out (`LeaseExpired`). The `GroupEvent::Leave` variant is only raised if some
  other implementation sends one.
- **`NewLeader` is never raised.** Call `leader()` yourself after each event.
- **Existing members don't answer a join.** A newcomer learns about each existing member only when that
  member's next keep-alive arrives, so its view can take up to one refresh period (13.5 s by default) to
  fill. Use `wait_for_view_size` to wait for it.
- **`MemberLiveliness::Manual` members always expire.** With `Manual`, no keep-alive task runs and there's
  no public method to send one, so other members drop you after one lease.
- `subscribe()` replaces the previous receiver: only the most recent caller gets events.

Examples: `zenoh-ext/examples/examples/z_member.rs` (prints events, view and leader) and `z_view_size.rs`
(`-g <group> -s <size> -t <timeout s> -i <id>`).

## PublicationCache (deprecated)

```rust
use zenoh_ext::SessionExt;
// Requires timestamping: config.timestamping.enabled = true
let cache = session.declare_publication_cache("sensors/**")
    .history(10)            // samples kept per key (default 1)
    .resources_limit(1000)  // max distinct keys (default unlimited)
    .queryable_suffix("cache")  // answer on "sensors/**/cache" instead of "sensors/**"
    .queryable_complete(true)
    .queryable_allowed_origin(Locality::Any)
    .await?;
```

- It declares a **session-local** subscriber (`allowed_origin(SessionLocal)`), so it only caches what
  **this session** publishes, not samples from the network.
- It fails to declare unless `timestamping/enabled` is true (`the 'timestamping' setting must be enabled`).
- Queries without wildcards are answered from the exact key. Queries with wildcards are matched against every
  cached key. A `_time=[…]` [time range](query.md) in the selector filters the samples.
- When `resources_limit` is reached, samples for new keys are dropped with an error log.

## QueryingSubscriber and FetchingSubscriber (deprecated)

`.querying()` on a subscriber builder makes a subscriber that runs a `get` at start-up and merges the
replies with live samples:

| Option | Default |
|---|---|
| `query_selector` | The subscriber's key expression |
| `query_target` | `All` |
| `query_consolidation` | `None` (keep every sample, so history isn't collapsed) |
| `query_accept_replies` | `MatchingQuery` |
| `query_timeout` | 10 s |

`.fetching(fetch_fn)` is the general form: you provide the function that fetches the initial samples (any
`get`, or something else). While a fetch is in progress, live samples are queued. When it ends, everything
is delivered **sorted by timestamp, with duplicate timestamps removed**. Samples without a timestamp are
delivered first. `FetchingSubscriber::fetch()` runs another fetch later.

On a **liveliness** subscriber builder, `.querying()` runs a liveliness `get` instead, which gives "current
tokens plus changes" (what `history(true)` on a liveliness subscriber does now).

## SubscriberForward

`subscriber.forward(sink)` forwards every sample from a FIFO-channel subscriber into any
`futures::Sink<Sample>`, such as a publisher wrapped as a sink. It's shorthand for
`subscriber.stream().map(Ok).forward(sink)`.

## Sources

- `zenoh-ext/src/lib.rs` (exports, feature gates), `group.rs`, `publication_cache.rs`,
  `querying_subscriber.rs`, `session_ext.rs`, `subscriber_ext.rs`
- `zenoh-ext/examples/examples/z_member.rs`, `z_view_size.rs`
