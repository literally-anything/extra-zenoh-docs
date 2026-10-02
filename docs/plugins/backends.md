# Storage backends

Backends plug into the [storage manager](storage-manager.md) as **volumes**. Each one is a dynamic library
named `zenoh_backend_<name>`, loaded from `backend_search_dirs` (or `__path__`). They're subject to the same
[binary compatibility rules](index.md#binary-compatibility) as plugins.

| Backend | Repo | Library | Typical volume entry | Notes |
|---|---|---|---|---|
| Memory | built in | — | `"memory"` | Always available, volatile |
| RocksDB | [zenoh-backend-rocksdb](https://github.com/eclipse-zenoh/zenoh-backend-rocksdb) | `zenoh_backend_rocksdb` | `volumes: { rocksdb: {} }`, storage `volume: { id: "rocksdb", dir: "example", create_db: true }` | Embedded, durable |
| Filesystem | [zenoh-backend-filesystem](https://github.com/eclipse-zenoh/zenoh-backend-filesystem) | `zenoh_backend_fs` | `volumes: { fs: {} }`, storage `volume: { id: "fs", dir: "example" }` | Each key becomes a file. Pairs well with the [web server plugin](ecosystem.md#web-server) |
| InfluxDB 1.x | [zenoh-backend-influxdb](https://github.com/eclipse-zenoh/zenoh-backend-influxdb) (`v1/`) | `zenoh_backend_influxdb` | `volumes: { influxdb: { url, private: {username, password} } }` | Time series, history |
| InfluxDB 2.x | same repo (`v2/`) | `zenoh_backend_influxdb2` | `volumes: { influxdb2: { url, private: {…} } }` | |
| Amazon S3 / compatible | [zenoh-backend-s3](https://github.com/eclipse-zenoh/zenoh-backend-s3) | `zenoh_backend_s3` | `volumes: { s3: { region, url, tls, private: { access_key, secret_key } } }`, storage `volume: { id: "s3", bucket, reuse_bucket, read_only, on_closure: "destroy_bucket" }` | Object storage |

Each repository has an `EXAMPLE_CONFIG.json5` with the full option list. All were at version 1.10.1 when
this page was written.

### Where the filesystem and RocksDB backends store data

A storage's `dir` must be **relative** (absolute paths and `..` are rejected) and is resolved against a root
directory, chosen in this order:

| Backend | 1. Environment variable | 2. Otherwise |
|---|---|---|
| Filesystem | `ZENOH_BACKEND_FS_ROOT` | `$ZENOH_HOME/zenoh_backend_fs` |
| RocksDB | `ZENOH_BACKEND_ROCKSDB_ROOT` | `$ZENOH_HOME/zenoh_backend_rocksdb` |

`ZENOH_HOME` defaults to `~/.zenoh`. These are the only places at 1.10.1 that read `ZENOH_HOME` (see
[Environment variables](../concepts/environment.md)). With the stock systemd unit (`ZENOH_HOME=/var/zenohd`),
a filesystem storage with `dir: "example"` ends up in `/var/zenohd/zenoh_backend_fs/example`.

## Writing a backend

Implement `zenoh_backend_traits::Volume` and `Storage`:

| Trait method | Purpose |
|---|---|
| `Volume::get_admin_status()` | JSON for the admin space |
| `Volume::get_capability()` | `Capability { persistence: Volatile \| Durable, history: Latest \| All }`: lets the manager choose trade-offs |
| `Volume::create_storage(StorageConfig)` | Create a storage instance |
| `Storage::put(key, payload, encoding, timestamp)` | Store a value. `key` is `None` when it equals `strip_prefix` exactly, and that entry must be stored too |
| `Storage::delete(key, timestamp)` | Delete |
| `Storage::get(key, parameters)` | Return the stored data for a key (and time range, if supported) |
| `Storage::get_all_entries()` | All `(key, latest timestamp)`, used for replication alignment |

Start from `plugins/zenoh-backend-example` in the main repo.

## Sources

- `plugins/zenoh-backend-traits/src/lib.rs`
- `plugins/zenoh-plugin-storage-manager/src/lib.rs` (`BACKEND_LIB_PREFIX = "zenoh_backend_"`)
- The backend repositories' `Cargo.toml` (`[lib] name`) and `EXAMPLE_CONFIG.json5`
