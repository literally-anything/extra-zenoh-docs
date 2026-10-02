# Admin space

Every Zenoh runtime can expose an **admin space**: a set of key expressions under
`@/<zid>/<mode>/**` that you can query with an ordinary `get` to inspect the node, and that accepts
`put`/`delete` under `config/**` to reconfigure plugins.

!!! info "Unstable"
    The upstream config says: *"Unstable: this configuration part works as advertised, but may change in a
    future release."*

## Enabling

```json5
adminspace: {
  enabled: true,                       // default false for library sessions
  permissions: { read: true, write: false },
},
```

| | Library session | `zenohd` |
|---|---|---|
| `adminspace/enabled` | `false` by default | **always forced `true`** (only `--cfg 'adminspace/enabled:false'` turns it off) |
| `permissions/read` | `true` | `true` (`--adminspace-permissions` changes it) |
| `permissions/write` | `false` | `false` (`--adminspace-permissions rw` enables it) |

With `read: false`, queries get an immediate final reply and the node logs
`Received GET on '…' but adminspace.permissions.read=false in configuration`.
With `write: false`, puts are ignored with a similar error.

## Root key

```
@/<zid>/<mode>
```

- `<zid>`: the node's Zenoh ID, as lowercase hex.
- `<mode>`: `router`, `peer` or `client`.

Because `@` is a [verbatim chunk](../concepts/key-expressions.md#verbatim-chunks), wildcards like `**`
**don't** reach the admin space. Write the `@` chunk explicitly:

| Query | Reaches |
|---|---|
| `@/*/router` | The root info of every router |
| `@/aaaa/router/**` | Everything on router `aaaa` |
| `@/*/*/subscriber/**` | Subscriber tables of every node with an admin space |
| `**` | **Nothing** in the admin space |

## Querying it

```bash
# Any Zenoh API:
session.get("@/*/router")

# Through the REST plugin (zenohd --rest-http-port 8000):
curl http://localhost:8000/@/aaaa/router
curl 'http://localhost:8000/@/*/router/subscriber/**'
```

Selector parameters (for example `?_stats` or `?compression=false;per_link=false`) use `;` as the separator.

## What's there

See the [key reference](reference.md) for every key, with real output.

| Key | Content |
|---|---|
| `@/<zid>/<mode>` | Node info: ZID, version, metadata, locators, sessions (with region, SHM, weights), plugins |
| `…/metrics` | OpenMetrics statistics (`stats` feature) |
| `…/linkstate/<region>` | Routing graph per region (Graphviz) |
| `…/subscriber/**`, `publisher/**`, `queryable/**`, `querier/**`, `token/**` | Entity tables |
| `…/route/successor/**` | Router next hops |
| `…/plugins/**` | Plugin status (with the `plugins` feature) |
| `…/status/plugins/**` | Data contributed by each plugin (version, config, storages, …) |
| `…/config/**` | **Write-only**: runtime config changes for `plugins/**` |

## Security

- Expose read-only (`r`) or nothing (`none`) on untrusted networks.
- Restrict `query` on `@/**` with [ACL](../security/access-control.md).
- `private` keys in plugin config are never shown in admin-space replies.

## Sources

- `zenoh/src/net/runtime/adminspace.rs`
- `commons/zenoh-config/src/lib.rs` (`AdminSpaceConf`, `PermissionsConf`)
- `zenohd/src/main.rs` (`--adminspace-permissions`, forced `enabled`)
