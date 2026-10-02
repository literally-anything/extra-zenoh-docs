# Scouting

Scouting finds Zenoh **nodes**. There are two mechanisms: **UDP multicast** for the local network, and
**gossip**, which spreads node information over existing connections.

## Multicast scouting

```json5
scouting: {
  timeout: 3000,          // client: give up finding a gateway after 3 s
  delay: 500,             // peer/router: how long open() waits for scouting
  multicast: {
    enabled: true,
    address: "224.0.0.224:7446",
    interface: "auto",    // or "eth0", "192.168.1.5", "eth0,wlan0"
    ttl: 1,
    listen: { router: true, peer: true, client: true },
    autoconnect: { router: [], peer: ["router", "peer", "client"], client: ["router", "peer", "client"] },
    autoconnect_strategy: { peer: { to_router: "always", to_peer: "always" } },
  },
},
```

### The protocol

- A node sends a **Scout** message to the multicast address. It contains a `what` matcher: the modes it's
  looking for.
- Every node with `listen` enabled whose mode matches `what` replies with a **Hello** **unicast** to the
  scouter. The Hello carries the node's ZID, mode and locators.
- Scouts are resent with backoff: first after **1 s**, doubling up to **8 s**, and repeating.
- Hello locators sent to a **loopback** scouter include loopback locators. Hellos to remote scouters leave
  them out.
- Hellos without locators are ignored (logged at debug).

### Interfaces

- `"auto"`: every up, non-loopback, multicast-capable interface. If there are none, `[::]` is used with a
  warning: `Unable to find active, non-loopback multicast interface. Will use [::].`
- A comma-separated list of names or IPs selects specific interfaces.
- For IPv4 groups the socket joins the group on each selected interface. For IPv6 it joins on interface 0.
- A TTL above 1 on an IPv6 group logs a warning: it may have no effect.

### What each mode does with it

| Mode | Sends scouts | Answers scouts | Connects to |
|---|---|---|---|
| Client | only when `connect/endpoints` is **empty** | if `listen.client` | the **first** node whose mode matches `autoconnect.client`. Fails after `scouting/timeout` |
| Peer | yes (if `autoconnect.peer` isn't empty) | if `listen.peer` | every node whose mode matches `autoconnect.peer`, subject to the strategy |
| Router | yes (if `autoconnect.router` isn't empty) | if `listen.router` | by default nobody (`autoconnect.router: []`) |

A client with no endpoints and scouting disabled fails with `No peer specified and multicast scouting deactivated!`.

### Autoconnect strategies

When two peers hear each other, both may try to connect, which wastes a connection. `autoconnect_strategy`
decides who dials:

| Strategy | Behaviour |
|---|---|
| `always` (default) | Always try. Duplicate connections are resolved and closed |
| `greater-zid` | Connect only if **my ZID > their ZID**. If both use it, exactly one side dials |

The strategy can depend on the remote's mode (`to_router`, `to_peer`) and on your own mode
([syntax](../configuration/mode-dependent-values.md#targetdependentvaluet-autoconnect-strategies)).

!!! warning "`greater-zid` and reachability"
    If the node with the greater ZID can't reach the other one (for example the other is behind NAT or on a
    private address), **neither** side connects. Use `greater-zid` only where every node can reach every
    other node.

### Avoiding duplicate connections

Before dialling a scouted node, Zenoh skips:

- locators that are already in `connect/endpoints` (the configured connector handles those);
- nodes it already has a transport with for the same priority range and reliability;
- nodes with a connection attempt already in progress.

## Gossip

```json5
scouting: {
  gossip: {
    enabled: true,
    multihop: false,
    target: { router: ["router", "peer"], peer: ["router", "peer"] },
    autoconnect: { router: [], peer: ["router", "peer", "client"], client: ["router", "peer", "client"] },
    autoconnect_strategy: { peer: { to_router: "always", to_peer: "always" } },
  },
},
```

Gossip lets nodes learn about other nodes **through** the nodes they're already connected to. It's what
lets a peer find the whole mesh from a single configured entry point, or work on networks where multicast
is blocked.

- Peers and routers send **link-state** messages to the neighbours that match `target`. Each one describes
  a node: ZID, mode, locators, sequence number, its links, and whether it's a region gateway.
- When a node learns about another node with locators, and `autoconnect` plus the strategy allow it, it
  connects.
- **Clients don't take part.** Configuring `"client"` in `gossip/target` is an error:
  `"client" is not allowed as gossip target`.

### `multihop`

| `multihop` | Behaviour | Cost |
|---|---|---|
| `false` (default) | Information goes **one hop**: you learn about your neighbours' neighbours | Low |
| `true` | Information spreads through the whole peer network (a full link-state graph) | More traffic, scales less well |

Turn on `multihop` when peers don't all have direct connectivity, so link-state routing through the peer
mesh is needed.

Internally, a north-bound peer hat uses a lightweight `Gossip` structure without multihop and a full
`Network` (link-state) with multihop. Peer hats on the south side of a gateway always use `Network`.

## Scouting from the API

`zenoh::scout(what, config)` runs multicast scouting without opening a session. It returns `Hello`s
(`zid()`, `whatami()`, `locators()`) through a handler. It uses the `scouting/multicast` address, interface
and TTL from the config you pass. See [API: Scouting](../api/scouting.md).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Peers on the same LAN don't find each other | Multicast blocked (Wi-Fi isolation, Docker bridge, cloud VPC). Configure `connect/endpoints`, or rely on gossip from one known peer |
| Duplicate connections in the admin space | Use `greater-zid` where every node can reach every other |
| Client fails after 3 s | No router or peer answered. Check the interface and multicast routing, or configure `connect/endpoints` |
| `Unable to bind UDP port 224.0.0.224:7446` | Another process or permission problem. Several Zenoh processes on one host can share the port (`SO_REUSEADDR` is set) |

## Sources

- `zenoh/src/net/runtime/orchestrator.rs` (`scout`, `responder`, `connect_first`, `autoconnect_all`, constants `SCOUT_*`)
- `zenoh/src/net/common.rs` (`AutoConnect::should_autoconnect`)
- `zenoh/src/net/protocol/gossip.rs`, `zenoh/src/net/routing/hat/peer/mod.rs`
- `commons/zenoh-config/src/defaults.rs` (`scouting`)
