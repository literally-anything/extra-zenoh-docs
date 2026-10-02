# Storage manager

The storage manager plugin (`zenoh_plugin_storage_manager`) turns `zenohd` into a store. Each **storage**
subscribes to a key expression, keeps the latest value per key in a **volume** (a backend), and answers
queries on it. Several storages on the same keys can be kept in sync as **replicas**.

## Configuration

```json5
plugins: {
  storage_manager: {
    __required__: false,
    backend_search_dirs: [],            // where to find zenoh_backend_* libraries (defaults to plugins_loading/search_dirs)
    volumes: {
      // "memory" always exists and needs no entry
      influxdb: {                       // loads libzenoh_backend_influxdb
        url: "https://influx.example",
        private: { username: "user1", password: "pw1" },   // hidden from the admin space
      },
      influxdb2: {
        backend: "influxdb",            // second volume using the same backend library
        __path__: ["/opt/zenoh/libzenoh_backend_influxdb.so"],
        url: "https://localhost:8086",
        private: { username: "user2", password: "pw2" },
      },
    },
    storages: {
      demo: { key_expr: "demo/memory/**", volume: "memory" },
      demo2: {
        key_expr: "demo/memory2/**",
        strip_prefix: "demo/memory2",
        volume: "memory",
        garbage_collection: { period: 30, lifespan: 86400 },
        replication: { interval: 10.0, sub_intervals: 5, hot: 6, warm: 30, propagation_delay: 250 },
      },
      demo3: { key_expr: "demo/memory3/**", volume: "memory", complete: true },
      influx_demo: {
        key_expr: "demo/influxdb/**",
        strip_prefix: "demo/influxdb",
        volume: { id: "influxdb", db: "example" },   // backend-specific storage options
      },
    },
  },
},
```

### Volumes

| Key | Meaning |
|---|---|
| `<name>` | Volume name. Unless `backend` says otherwise, the library `zenoh_backend_<name>` is loaded |
| `backend` | Backend library to load (lets several volumes share one backend) |
| `__path__` | Explicit library path(s) |
| `__required__` | Fail if the backend can't be loaded |
| `private` | Credentials etc., hidden from the admin space |
| other keys | Passed to the backend |

The `memory` volume is built in (`__path__` shows `__static_lib__`).

### Storages

| Key | Default | Meaning |
|---|---|---|
| `key_expr` | **required** | Keys this storage subscribes to and answers queries for |
| `volume` | **required** | A volume name, or `{ id: "<volume>", …backend options… }` |
| `strip_prefix` | none | Prefix removed from keys before storing. Must be a **prefix of `key_expr`** and contain **no wildcards** |
| `complete` | `false` | Declare the queryable as **complete** for `key_expr`: "I hold every key in this set". Queries with target `AllComplete` rely on it |
| `garbage_collection.period` | 30 s | How often metadata is garbage-collected |
| `garbage_collection.lifespan` | 86400 s | Metadata (tombstones, wildcard updates) older than this is removed |
| `replication` | none | Turns on replica alignment (below) |

!!! warning
    `strip_prefix` and **all** `replication` values must be **identical on every replica** of a storage,
    otherwise replicas never converge.

## Behaviour

- Each storage declares a **subscriber** and a **queryable** (`complete` as configured) on `key_expr`.
- Every stored value has a **timestamp**. Samples that arrive without one get one from the plugin's session.
  Routers timestamp data by default ([Timestamping](../configuration/session-behaviour.md#timestamping)).
- Newer timestamps win. Deletes are kept as **tombstones**, and puts or deletes on **wildcard** keys are
  remembered so that later, older updates are overridden correctly. This metadata is what
  `garbage_collection` cleans up.
- Queries are answered from the volume, honouring the selector and time range when the backend supports it.

## Replication

With `replication` set on storages that share the same `key_expr`, the replicas exchange **digests** and
align their contents:

| Key | Default | Meaning |
|---|---|---|
| `interval` | 10.0 s | How often digests are computed and published. Also how long replicas may diverge |
| `sub_intervals` | 5 | Subdivisions of an interval in the fingerprint. Higher means bigger digests but cheaper alignment |
| `hot` | 6 | Intervals in the "hot" era (6 × 10 s = the last minute) |
| `warm` | 30 | Intervals in the "warm" era (30 × 10 s = 5 minutes) |
| `propagation_delay` | 250 ms | Expected time for a publication to reach the storage. Digests are computed at `n × interval + propagation_delay`. Must be **less than half** of `interval` |

All samples stored in replicas must be timestamped.

## Runtime changes

The storage manager accepts runtime configuration changes ([Dynamic changes](../configuration/dynamic-changes.md)):
adding a storage or volume starts it, and removing one stops it. Verified:

```bash
curl -X PUT -H 'content-type: application/json' -d '{"key_expr":"demo3/**","volume":"memory"}' \
  http://localhost:8000/@/local/router/config/plugins/storage_manager/storages/demo3
curl 'http://localhost:8000/@/local/router/status/plugins/storage_manager/storages/**'
# → …/storages/demo  {"key_expr":"demo/**","volume":"memory"}
# → …/storages/demo3 {"key_expr":"demo3/**","volume":"memory"}
```

## Admin space

- `@/<zid>/router/status/plugins/storage_manager/storages/<name>`: each storage's config/status
- `…/volumes/<name>` and `…/volumes/<name>/__path__`
- `@/<zid>/router/plugins/storage_manager/<backend>`: backend plugin status

## Sources

- `plugins/zenoh-plugin-storage-manager/src/` (`lib.rs`, `storages_mgt/service.rs`, `replication/`)
- `plugins/zenoh-backend-traits/src/config.rs` (`StorageConfig`, `ReplicaConfig`, `GarbageCollectionConfig`)
- `DEFAULT_CONFIG.json5` (`plugins.storage_manager` example)
