# Regions & gateways

Zenoh 1.10 organises routing into **regions**. Every node is a **gateway** that keeps one routing table
(HAT) per region and moves data between them. Regions let you build hierarchies (site → building → floor),
isolate subsystems, and decide which remote nodes count as "below" a node.

This page is built from `zenoh/src/net/runtime/region.rs`, `routing/gateway.rs` and
`commons/zenoh-config/src/gateway.rs`, and checked with live `zenohd` 1.10.1 routers (output below).

## The model

| Region | Who's in it | HAT |
|---|---|---|
| `north` | Nodes "above" or beside this node, using the node's own mode | router / peer / client HAT |
| `south:<id>:<mode>` | Remote nodes placed **below** this node, in subregion `<id>`, grouped by **their** mode | router / peer / broker HAT |
| `local` | This node's own sessions (applications, plugins, admin space) | broker HAT |

Each link to a remote node also has a **bound**, `north` or `south`, which says which side of the
relationship it's on.

### Which regions exist

From `GatewayBuilder::build`:

| `gateway.south` | Router | Peer | Client |
|---|---|---|---|
| `"auto"` (default) | `north`, `south:0:client`, `south:0:peer`, `local` | `north`, `south:0:client`, `local` | `north`, `local` |
| Custom list of N subregions | `north`, plus `south:i:client`, `south:i:peer`, `south:i:router` for each i < N, plus `local` | same | same |

## The `auto` preset

With the default `gateway: { south: "auto" }`:

| This node | Remote | Remote is placed in |
|---|---|---|
| router | router | `north` (router backbone) |
| router | peer or client | `south:0:<mode>` |
| peer | peer | `north` (peer mesh) |
| peer | client | `south:0:client` |
| peer or client | router | `north` |
| client | peer | `north` |
| client | client | **invalid**: `North-north client-client configuration (invalid)` |

That matches the traditional Zenoh picture: routers in a backbone, peers and clients hanging off routers,
clients hanging off peers.

## Custom subregions

```json5
{
  mode: "router",
  region_name: "hq",                      // this node's own region name (1–32 bytes UTF-8)
  gateway: {
    south: [
      // subregion 0: anything that says it's in region "site-b"
      { filters: [ { region_names: ["site-b"] } ] },
      // subregion 1: peers or clients reached through eth1
      { filters: [ { modes: ["peer", "client"], interfaces: ["eth1"] } ] },
      // subregion 2: everything EXCEPT one ZID
      { filters: [ { zids: ["abcd"], negated: true } ] },
    ],
  },
}
```

How a remote is placed:

1. Subregions are tried **in order**. The remote goes into the **first** subregion whose filters match:
   `south:<index>:<remote mode>`.
2. If none matches, the remote is in `north`.

How filters match (`is_match`):

| Field | The remote matches if… |
|---|---|
| `filters` missing | always |
| `filters: []` | never |
| list of filters | **any** filter matches (OR) |
| within one filter | **every** field given matches (AND) |
| `modes` | its mode is in the matcher |
| `zids` | its ZID is in the list |
| `interfaces` | **all** local interfaces its links use are in the list |
| `region_names` | it announced a `region_name` that's in the list (no `region_name` = no match) |
| `negated: true` | inverts that filter's result |

`region_name` is exchanged during transport establishment (the `region_name` extension).

### Rules and errors

The final placement combines this node's view with the bound the remote announces. These combinations are
rejected, and the transport fails to open:

| Situation | Error |
|---|---|
| Both nodes place each other **south** | `South-south configuration (invalid)` |
| Both place each other north explicitly, with different modes | `North-north <mode>-<mode> configuration (invalid)` |
| A remote **router** matches a subregion of a node that isn't a router | `Router regions cannot be subregions of non-router regions (unsupported)` |
| The remote's custom config wants it north of us, but our auto preset puts it south | `Remote's custom configuration conflicts with auto preset` |
| Our custom config puts the remote north, but its auto preset puts us south | `Remote's auto preset conflicts with custom configuration` |

### Multicast

Multicast transports only work in two places: `north` for peers, and `south:0:peer` for routers. A client
on a multicast transport fails with `Multicast is only supported in north peer & south router regions`.

## Verified example

Three routers: `aaaa`, `bbbb` (with `region_name: "site-b"`) connected to `aaaa`, and `dddd` (`region_name: "hq"`)
connected to `bbbb` with the custom gateway below. A peer `cccc` is also connected to `aaaa`.

```json5
// dddd
{ mode: "router", id: "dddd", region_name: "hq",
  connect: { endpoints: ["tcp/127.0.0.1:17448"] },   // bbbb
  gateway: { south: [ { filters: [ { region_names: ["site-b"] } ] } ] } }
```

What the admin space shows (`@/<zid>/router` → `sessions[].region`, and `linkstate/*` keys):

```text
dddd sees  bbbb  router  south:0:router
dddd linkstate keys: north, south:0:router, south:0:peer

aaaa sees  bbbb  router  north           (auto preset: router-router)
aaaa sees  cccc  peer    south:0:peer    (auto preset: peer south of router)
aaaa linkstate keys: north, south:0:peer
```

Data and queries cross the region boundaries as you'd expect: a `GET demo/**` sent through `dddd`'s REST
plugin returned data from the storage on `aaaa`, and a `PUT` through `dddd` reached that storage.

## When to use custom regions

- **Hierarchical sites**: an HQ router treats each site's router as a south subregion, so site-local
  traffic stays local and only data someone has asked for crosses to the site.
- **Separating subsystems** on one router by interface (`interfaces: ["eth1"]`) or by `region_name`.
- **Gateways for peer meshes**: a router with a `south:*:peer` region acts as the gateway for a whole peer mesh.

## Admin space

- `@/<zid>/<mode>` → `sessions[].region`: the region of each remote (`north`, `south:0:peer`, …).
- `@/<zid>/<mode>/linkstate/<region>`: one link-state graph per peer or router region.

## Sources

- `commons/zenoh-config/src/gateway.rs` (`GatewayConf`, filters, `negated`)
- `commons/zenoh-protocol/src/core/region.rs` (`Region`, `Bound`, string forms)
- `zenoh/src/net/runtime/region.rs` (`compute_region_of`, `compute_auto_region`, `is_match`, multicast rules)
- `zenoh/src/net/routing/gateway.rs` (regions created per mode and preset)
- `io/zenoh-transport/src/unicast/establishment/ext/region_name.rs`
