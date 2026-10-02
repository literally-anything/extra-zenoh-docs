# Query / queryable / querier

Query/reply is Zenoh's request–response paradigm. A **queryable** serves data or computes answers for a
key expression, and a **get** or a **querier** asks for them.

## Selectors

A selector is a key expression plus parameters, written like a URL:

```
robot/*/battery?_time=[now(-1h)..];format=json
```

- Parameters follow the first `?`, are separated by **`;`**, and use `name=value` (a name with no `=` has
  an empty value). Percent-encode special characters.
- Names starting with a non-alphanumeric character are **reserved** for Zenoh:

| Parameter | Meaning |
|---|---|
| `_time` :material-flask: | Only values dated within a time range (see below). Storages honour it |
| `_anyke` :material-flask: | Accept replies on keys outside the query's key expression (set by `accept_replies(ReplyKeyExpr::Any)`) |

### Time-range syntax (`_time`)

- Range form: `[start..end]`. Either side may be omitted, and `]`/`[` facing outward means exclusive.
  Example: `[now(-1h)..]` means the last hour.
- Duration form: `[start;duration]`, for example `[2026-10-01T00:00:00Z;1d]`.
- Instants: RFC 3339 UTC, or `now(<±offset>)`. Duration units: `u`, `ms`, `s`, `m`, `h`, `d`, `w`.

## Get

```rust
let replies = session.get("robot/*/battery?_time=[now(-1h)..]")
    .target(QueryTarget::All)
    .consolidation(ConsolidationMode::None)
    .timeout(Duration::from_secs(2))
    .payload("optional request body")
    .attachment("…")
    .await?;
while let Ok(reply) = replies.recv_async().await {
    match reply.result() {
        Ok(sample) => println!("{} = {:?}", sample.key_expr(), sample.payload()),
        Err(err)   => println!("error: {:?}", err.payload()),
    }
}
```

| Option | Default | Meaning |
|---|---|---|
| `target` | `BestMatching` | `BestMatching`: if a matching queryable declared **complete** is known, only that one (the first found) gets the query, otherwise it behaves like `All`. `All`: every matching queryable. `AllComplete`: only matching queryables declared `complete` |
| `consolidation` | `Auto` | `Auto` becomes `Latest`, or `None` if the selector has `_time` |
| `timeout` | `queries_default_timeout` (10 s) | When to stop waiting. The channel then closes |
| `payload`, `encoding` | none | Request body |
| `attachment` | none | Extra metadata |
| `allowed_destination` | `Any` | Only local or only remote queryables |
| `accept_replies` :material-flask: | `MatchingQuery` | `Any` allows replies on non-matching keys |
| `congestion_control`, `priority`, `express` | `Block`, `Data`, `false` | QoS of the query. Replies inherit it by default |

### Consolidation

| Mode | Behaviour |
|---|---|
| `None` | Every reply is passed on, so you may see several samples for the same key |
| `Monotonic` | Forwarded straight away, but a sample is dropped if a reply with the same key and an equal or newer timestamp was already forwarded. Lower latency |
| `Latest` | Replies are held back and only the newest sample per key is delivered, at the end |
| `Auto` | `Latest`, or `None` when `_time` is present |

## Queryable

```rust
let queryable = session.declare_queryable("robot/arm/state")
    .complete(true)
    .await?;
while let Ok(query) = queryable.recv_async().await {
    let params = query.parameters();
    query.reply("robot/arm/state", "{\"angle\":42}")
        .encoding(Encoding::APPLICATION_JSON)
        .await?;
    // or: query.reply_err("not ready").await?;
    // or: query.reply_del("robot/arm/state").await?;
}
```

| `Query` accessor | Meaning |
|---|---|
| `selector()`, `key_expr()`, `parameters()` | What was asked |
| `payload()`, `encoding()`, `attachment()` | Request body |
| `source_info()` :material-flask: | Who asked |
| `accepts_replies()` :material-flask: | Whether non-matching reply keys are allowed |

- Reply keys must **intersect** the query's key expression unless the querier accepts `Any`.
- `complete(true)` declares that this queryable has **every** key matching its expression. `BestMatching`
  then sends the query to it alone, and `AllComplete` only targets complete queryables.
- Final delivery (`ResponseFinal`) is sent when the `Query` object is dropped. Keep it alive until you've
  sent all your replies.

## Querier

A querier is to `get` what a publisher is to `put`: a declared, reusable sender with fixed options and
[matching](../discovery/matching.md) support.

```rust
let querier = session.declare_querier("robot/*/state")
    .target(QueryTarget::All)
    .timeout(Duration::from_secs(1))
    .await?;
let replies = querier.get().parameters("verbose=1").await?;
let listener = querier.matching_listener().await?;   // any matching queryables?
```

## Sources

- `zenoh/src/api/query.rs`, `queryable.rs`, `querier.rs`, `selector.rs`, `builders/query.rs`, `builders/reply.rs`
- `commons/zenoh-protocol/src/network/request.rs` (`QueryTarget`), `zenoh/query.rs` (`ConsolidationMode`)
- `commons/zenoh-util/src/time_range.rs`
- `zenoh/src/api/session.rs` (`Auto` resolution)
