# Matching

Matching tells the **active** side of a conversation whether anyone is on the other end:

- a **publisher** whether at least one **subscriber** matches its key expression;
- a **querier** whether at least one **queryable** matches its key expression **and** query target.

Use it to skip expensive work (encoding a camera frame, say) when nobody is listening.

## API

```rust
let publisher = session.declare_publisher("camera/front").await?;

// One-off check
let status = publisher.matching_status().await?;
if status.matching() { /* someone is subscribed */ }

// Get notified of changes
let listener = publisher.matching_listener().await?;
while let Ok(status) = listener.recv_async().await {
    if status.matching() { start_capture(); } else { stop_capture(); }
}
```

`Querier::matching_status()` and `Querier::matching_listener()` work the same way. A matching listener
works like a subscriber, but instead of samples it yields a `MatchingStatus` each time the status
**changes**: when the first match appears and when the last one goes away.

Matching is a **stable** API in 1.10 (no `unstable` feature needed).

## How it works

Declaring a publisher or querier sends a `CurrentFuture` [interest](interests.md) for subscribers or
queryables on its key expression. The declarations that come back (and later updates) drive the matching
status. A matching listener is notified when the set goes from empty to non-empty or back.

Local entities in the same session count as matches unless the publisher's `allowed_destination`
(:material-flask: unstable) excludes them.

## Bindings

Matching listeners are available in most bindings. See the [API matrix](../api/index.md) for each one.

## Sources

- `zenoh/src/api/matching.rs`, `zenoh/src/api/builders/matching_listener.rs`
- `zenoh/src/lib.rs` (`pub mod matching`)
