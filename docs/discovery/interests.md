# Interests & declarations

Once two nodes are connected, they tell each other about **entities**: subscribers, queryables, liveliness
tokens and declared key expressions. This happens with **declarations**, and since Zenoh 1.0 most of them
are driven by **interests**.

## Declarations

| Declaration | Sent when |
|---|---|
| `DeclareKeyExpr` | A key expression is mapped to a numeric ID, which saves bandwidth |
| `DeclareSubscriber` / `UndeclareSubscriber` | A subscriber appears or disappears somewhere behind this face |
| `DeclareQueryable` / `UndeclareQueryable` | Same for queryables (with `complete` info) |
| `DeclareToken` / `UndeclareToken` | Liveliness tokens |
| `DeclareFinal` | Marks the end of the declarations that answer an interest |

Routers and peers aggregate and forward declarations according to their routing algorithm (see
[Routing](../topology/routing.md)). You can see what a router knows under `@/<zid>/router/subscriber/**`,
`queryable/**` and `token/**` ([Admin space](../admin-space/reference.md)).

## Interests

An **Interest** asks the other side for declarations of some kind, optionally limited to a key expression:

| Mode | Meaning |
|---|---|
| `Current` | Send me the matching declarations that exist **now**, then `DeclareFinal` |
| `Future` | Send me matching declarations and undeclarations **from now on** |
| `CurrentFuture` | Both |
| `Final` | Stop sending (cancels a Future/CurrentFuture interest) |

Interest options select the kinds: key expressions (`K`), subscribers (`S`), queryables (`Q`), tokens (`T`),
whether the key-expression restriction applies (`R`), whether IDs may be aggregated (`A`), and so on.

### Who sends interests

- A **client** sends interests to its gateway, so it only learns about the declarations it needs, instead
  of being flooded with every declaration in the network.
- **Declaring a publisher** sends a `CurrentFuture` interest for subscribers (with key expressions) on the
  publisher's key. This feeds [matching](matching.md) and lets routers avoid sending data nobody wants.
  Undeclaring the publisher sends a `Final`.
- **Declaring a querier** does the same for queryables.
- **Liveliness subscribers** with `history(true)` and liveliness `get` use token interests.
- Between **peers**, when a connection opens, each side behaves as if it had received a `CurrentFuture`
  interest with ID 0 and replies with its declarations followed by a `DeclareFinal` (the "initial interest").

## `routing/interests/timeout`

```json5
routing: { interests: { timeout: 10000 } },   // ms
```

When a node forwards a `Current`/`CurrentFuture` interest, it waits for the `DeclareFinal`. If that doesn't
arrive within the timeout, the node gives up waiting, logs a warning, and finalises the interest towards the
requester:

```
WARN ... Didn't receive DeclareFinal for interest ...: Timeout(10s)!
```

When that happens, discovery may be **incomplete**: some subscribers, queryables or tokens may not be known
yet, and publications, queries or liveliness information can be lost until they arrive.

## Session open and declarations

`open/return_conditions/declares` (default `true`) makes `zenoh::open()` wait for the initial declarations
from connected peers before returning. With `false`, the session returns sooner, but early puts may go out
without knowing who's subscribed (more traffic) or reach nobody.

## Sources

- `commons/zenoh-protocol/src/network/interest.rs` (modes, flow diagrams, options)
- `zenoh/src/api/session.rs` (interests sent by publishers, queriers, liveliness)
- `zenoh/src/net/routing/dispatcher/interests.rs` (timeout and finalisation)
- `zenoh/src/net/routing/hat/peer/mod.rs` (initial interest)
