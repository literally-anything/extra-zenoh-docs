# Routing

Routing in Zenoh is done by **HATs** (Hierarchical Algorithm Tables). There are four HAT
implementations. Since 1.10, every node runs **one HAT per region** (see [Regions](regions.md)), picked
from the region's bound and mode:

| Region bound | Mode | HAT | Algorithm |
|---|---|---|---|
| North | client | `client` | Forward everything to the single gateway |
| North | peer | `peer` | Gossip / link-state for discovery, direct delivery to mesh neighbours |
| North | router | `router` | Full link-state, shortest-path trees |
| South | client | `broker` | The gateway serves clients like a broker (also used for the node's own **local** sessions) |
| South | peer | `peer` (with `Network`) | Peer subsystem south of a router |
| South | router | `router` | Router subsystem south of another router (custom regions only) |

## Router link-state

Routers in the same region exchange **link-state** messages describing their links. Each router builds the
graph and computes, for every router, a **shortest-path tree** (Bellman-Ford, in `protocol/network.rs`).
Declarations and data follow these trees, so each message crosses each link at most once.

- Trees are recomputed after topology changes, debounced by `TREES_COMPUTATION_DELAY_MS` (100 ms).
- When a link is lost, routers that are no longer reachable are removed from the graph.

### Link weights

```json5
routing: {
  router: {
    linkstate: {
      transport_weights: [
        { dst_zid: "bbbb", weight: 10 },    // prefer this link
        { dst_zid: "cccc", weight: 500 },   // avoid this link
      ],
    },
  },
},
```

- A link's weight is 100 by default. If one end sets a weight, that weight is used. If both ends set one,
  the **greater** weight is used.
- `weight` is a non-zero u16.
- To break ties, every edge weight is multiplied by `1 + 0.01 × hash(zid₁, zid₂)/u32::MAX`. That's up to
  **+1%**, deterministic for a given pair of ZIDs, so equal-cost paths are chosen the same way on every
  router. This is why the link-state graph in the admin space shows values like `100.53`.
- The effective weights appear in the admin root under `sessions[].weight` as `{ src_weight, dst_weight, actual_weight }`.

## Peer routing

- Peers in the north region **don't forward** to other peers. Each publisher's session sends straight to
  each subscriber's peer, so the mesh must be complete (see [Topology](index.md)).
- `scouting/gossip/multihop: true` spreads node information (locators) across more than one hop so a full
  mesh can form, but it doesn't add forwarding.
- Peers are gateways for clients south of them (a client connected to a peer reaches the peer's mesh).

## Clients

A client sends all its declarations, data and queries to its one gateway and receives what the gateway
sends back. It uses [interests](../discovery/interests.md) so it only receives declarations it needs.

## Seeing the routing state

All in the [admin space](../admin-space/reference.md):

| Key | Shows |
|---|---|
| `@/<zid>/router/linkstate/<region>` | Graphviz `dot` of the region's link-state graph (`north`, `south:0:peer`, …) |
| `@/<zid>/router/route/successor/src/<zid>/dst/<zid>` | Next hop from `src` towards `dst` (router regions) |
| `@/<zid>/router/subscriber/**`, `queryable/**`, `token/**` | Known entities with the routers, peers and clients they come from |

Example from a two-router setup (`aaaa` ↔ `bbbb`):

```text
@/aaaa/router/linkstate/north
graph {
    0 [ label = "aaaa" ]
    1 [ label = "bbbb" ]
    1 -- 0 [ label = "100.5317599888732" ]
}

@/aaaa/router/route/successor/src/aaaa/dst/bbbb   →  "bbbb"
```

## Deprecated options

`routing/peer/mode` (`peer_to_peer` / `linkstate`), `routing/peer/linkstate` and
`routing/router/peers_failover_brokering` are accepted for compatibility, log a deprecation warning, and
**do nothing**. Peer link-state routing was replaced by the region model.

## Sources

- `zenoh/src/net/routing/gateway.rs` (HAT selection)
- `zenoh/src/net/routing/hat/router/` (trees, `TREES_COMPUTATION_DELAY_MS`), `hat/peer/`
- `zenoh/src/net/protocol/network.rs` (graph, Bellman-Ford, edge weights), `linkstate.rs` (`DEFAULT_LINK_WEIGHT = 100`)
- `zenoh/src/net/runtime/adminspace.rs` (`linkstate_data`, `route_successor`)
