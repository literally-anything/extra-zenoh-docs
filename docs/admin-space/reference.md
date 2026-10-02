# Admin space key reference

Every handler registered in `AdminSpace::start` (`zenoh/src/net/runtime/adminspace.rs`). The outputs were
captured from `zenohd` 1.10.1 routers `aaaa` and `bbbb` (connected, with REST and an in-memory storage on
`aaaa`, and a storage on `bbbb`) plus a peer `cccc` connected to `aaaa`.

## `@/<zid>/<mode>`: node info

Encoding `application/json`.

```json
{
  "zid": "aaaa",
  "version": "v1.10.1-173b1220c2ab59cc22c82bfc6c95ac9971ff213b built with rustc 1.97.1 (8bab26f4f 2026-07-14)",
  "metadata": { "location": "lab", "name": "router-1" },
  "locators": ["tcp/127.0.0.1:17447"],
  "sessions": [
    {
      "peer": "bbbb",
      "whatami": "router",
      "region": "north",
      "shm": true,
      "links": [ { "src": "tcp/127.0.0.1:17447", "dst": "tcp/127.0.0.1:48228" } ],
      "weight": { "actual_weight": 100, "dst_weight": null, "src_weight": null }
    },
    {
      "peer": "cccc",
      "whatami": "peer",
      "region": "south:0:peer",
      "shm": true,
      "links": [ { "src": "tcp/127.0.0.1:17447", "dst": "tcp/127.0.0.1:34686" } ],
      "weight": null
    }
  ],
  "plugins": {
    "rest": { "name": "rest", "path": ".../libzenoh_plugin_rest.so" },
    "storage_manager": { "name": "storage_manager", "path": ".../libzenoh_plugin_storage_manager.so" }
  }
}
```

| Field | Meaning |
|---|---|
| `metadata` | The `metadata` config value, unchanged |
| `locators` | Listening locators |
| `sessions[]` | One entry per unicast transport: remote `peer` ZID, `whatami`, [`region`](../topology/regions.md), `shm` negotiated, `links` (src/dst), and link-state `weight` (routers only) |
| Multicast `sessions[]` | `peer`, `whatami`, `group`, `links` |
| `plugins` | Loaded plugins (name and library path) |

With the `stats` feature, add `?_stats` to the selector to merge statistics into this JSON.

## `…/metrics`

OpenMetrics text (`application/openmetrics-text; version=1.0.0; charset=utf-8`), gzip by default.
Parameters: `compression`, `per_transport`, `per_link`, `disconnected`, `per_key`, `descriptors`.
Without the `stats` feature only `zenoh_build_info` is returned. See [Statistics](../configuration/stats.md).

## `…/linkstate/<region>`

One `text/plain` Graphviz graph per **peer or router region** (`north`, `south:0:peer`, …):

```text
@/aaaa/router/linkstate/north
graph {
    0 [ label = "aaaa" ]
    1 [ label = "bbbb" ]
    1 -- 0 [ label = "100.5317599888732" ]
}

@/aaaa/router/linkstate/south:0:peer
graph {
    0 [ label = "aaaa" ]
    1 [ label = "cccc" ]
}
```

Edge labels are link weights, including the tie-break jitter (see [Routing](../topology/routing.md#link-weights)).
Paste the output into any Graphviz viewer.

## `…/subscriber/**`, `…/publisher/**`, `…/queryable/**`, `…/querier/**`, `…/token/**`

One reply per resource, keyed by the resource's key expression. The value lists **where the entity is
declared**, grouped by mode:

```text
@/aaaa/router/subscriber/demo/**   {"clients":["aaaa"],"peers":[],"routers":["aaaa"]}
@/aaaa/router/subscriber/demo2/**  {"clients":[],"peers":[],"routers":["bbbb"]}
@/aaaa/router/queryable/demo/**    {"clients":["aaaa"],"peers":[],"routers":["aaaa"]}
```

The node's **own** sessions (plugins, admin space, local apps) appear under `clients` with the node's own
ZID, because they live in the [local region](../topology/regions.md#the-model), which is served like
clients.

You can narrow the query: `@/aaaa/router/subscriber/demo/**` returns only resources under `demo/`.

## `…/route/successor/**`

Router regions only. For each pair of routers, the next hop from `src` towards `dst`:

```text
@/aaaa/router/route/successor/src/aaaa/dst/bbbb   "bbbb"
@/aaaa/router/route/successor/src/bbbb/dst/bbbb   "bbbb"
```

Querying the exact key `…/route/successor/src/<zid>/dst/<zid>` takes a shortcut and doesn't build the full table.

## `…/plugins/**`

With the `plugins` feature (always in `zenohd`). Status per plugin:

```json
@/aaaa/router/plugins/storage_manager
{ "id": "storage_manager", "name": "storage_manager", "version": "1.10.1", "long_version": "v173b122",
  "path": ".../libzenoh_plugin_storage_manager.so", "state": "Started", "report": { "level": "Info" } }

@/aaaa/router/plugins/memory
{ "id": "memory", "name": "storage_manager/memory", "path": "__static_lib__", "state": "Started", ... }
```

Storage backends (volumes) show up as plugins too (`storage_manager/memory`).

## `…/status/plugins/**`

Each running plugin answers queries under `status/plugins/<id>/**` (its `adminspace_getter`). The admin
space itself adds `status/plugins/<id>/__path__`. Captured from the REST and storage-manager plugins:

```text
@/aaaa/router/status/plugins/rest/__path__                         ".../libzenoh_plugin_rest.so"
@/aaaa/router/status/plugins/rest/version                          "v173b122"
@/aaaa/router/status/plugins/rest/port                             {"http_port":"127.0.0.1:18000","work_thread_num":2,"max_block_thread_num":50,...}
@/aaaa/router/status/plugins/storage_manager/version               "1.10.1"
@/aaaa/router/status/plugins/storage_manager/storages/demo         {"key_expr":"demo/**","volume":"memory"}
@/aaaa/router/status/plugins/storage_manager/volumes/memory        {"__required__":false}
@/aaaa/router/status/plugins/storage_manager/volumes/memory/__path__  "__static_lib__"
```

If a plugin panics while answering, the panic is caught and logged
(`Plugin <id> panicked while responding to …`), and the admin space keeps working.

## `…/config/**` (write)

Not readable. A `put` or `delete` here changes the runtime configuration, but only under `plugins/`. See
[Dynamic changes](../configuration/dynamic-changes.md).

## Session-local admin space: `@/<zid>/session/**`

Separately from the runtime admin space above, **every session** (since 1.10, enabled whatever the
`adminspace` config says) declares a queryable on `@/<own zid>/session/**` with locality
**`SessionLocal`**. Only the session itself can query it, and nothing on the network can see it. It also
**publishes** (session-locally) a sample each time a transport or link opens or closes:

| Key | Value (JSON) |
|---|---|
| `@/<zid>/session/transport/unicast/<peer_zid>` | `{ "zid", "whatami", "is_qos", "is_shm" }` (`is_shm` only with `shared-memory`) |
| `@/<zid>/session/transport/multicast/<peer_zid>` | same |
| `@/<zid>/session/transport/<kind>/<peer_zid>/link/<hash>` | `{ "src", "dst", "group", "mtu", "is_streamed", "interfaces", "auth_identifier", "priorities"?, "reliability"? }` |

A `Put` means opened, and a `Delete` (empty payload) means closed. An application can therefore watch its
own connectivity with a plain subscriber on `@/<own zid>/session/transport/**`, without the unstable event
API. A second queryable, under
`@adv/pub/<zid>/_/_/@/@/<zid>/session/**`, imitates an [advanced publisher](../api/advanced-pubsub.md), so an
`AdvancedSubscriber` with history also receives the current state when it starts.

## Constants exported by the API

With the `internal` feature (meant for bindings), `zenoh` re-exports the key chunks used above: `KE_AT` (`@`),
`KE_ADV_PREFIX` (`@adv`), `KE_PUB` (`pub`), `KE_SUB` (`sub`), `KE_EMPTY` (`_`), `KE_STAR` (`*`), `KE_STARSTAR` (`**`).

## Sources

- `zenoh/src/net/runtime/adminspace.rs` (`local_data`, `metrics`, `linkstate_data`, `resources_data`,
  `route_successor`, `plugins_data`, `plugins_status`)
- `zenoh/src/api/admin.rs` (session-local admin space)
