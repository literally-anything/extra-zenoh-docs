# zenohd (router binary)

`zenohd` is the stand-alone Zenoh router. It's a thin wrapper: it builds a `Config` from the command line,
calls `zenoh::open(config)` and parks the main thread. Everything it does is something a library session
could do too. What it adds is plugin loading and some defaults that suit a router.

## Command-line flags

| Flag | Value | Effect on the config |
|---|---|---|
| `-c`, `--config` | `PATH` | Load the config file (`.json5`, `.json`, `.yaml`/`.yml`; `.toml` because zenohd is built with `unstable`). Ignored if `--cfg ':{...}'` gives a whole inline config. |
| `-l`, `--listen` | `ENDPOINT` (repeatable) | **Replaces** `listen/endpoints` with the given list |
| `-e`, `--connect` | `ENDPOINT` (repeatable) | **Replaces** `connect/endpoints` with the given list. Each value is a single endpoint. For `{strategy, locators}` groups (see [Endpoints](../configuration/endpoints.md#endpoint-groups)), use the config file or `--cfg` |
| `-i`, `--id` | hex string | Sets `id`. Must be unique and at most 16 bytes (32 hex chars) |
| `-P`, `--plugin` | `NAME` or `NAME:PATH` (repeatable) | Sets `plugins/NAME/__required__ = true` and, with a path, `plugins/NAME/__path__ = "PATH"` |
| `--plugin-search-dir` | `PATH` (repeatable) | **Replaces** `plugins_loading/search_dirs` with these paths |
| `--no-timestamp` | — | `timestamping/enabled = false` for all modes |
| `--no-multicast-scouting` | — | `scouting/multicast/enabled = false` |
| `--rest-http-port` | port, `IP:PORT`, or `none` | Sets `plugins/rest/http_port` and `plugins/rest/__required__ = true`. `none` leaves the config alone |
| `--cfg` | `KEY:VALUE` (repeatable) | Inserts the JSON5 `VALUE` at config path `KEY` (a leading `/` is stripped). An empty key (`--cfg ':{...}'`) replaces the whole config |
| `--adminspace-permissions` | `r`, `w`, `rw`, `none` | Sets `adminspace/permissions/{read,write}` |

## Order of application

From `config_from_args` in `zenohd/src/main.rs`:

1. Start from `--cfg ':{...}'` if given, otherwise `-c FILE`, otherwise `Config::default()`.
2. If `mode` isn't set, set it to `router`.
3. Apply `--id` and `--rest-http-port`.
4. **Always** set `adminspace/enabled = true` and `plugins_loading/enabled = true`.
5. Apply `--plugin-search-dir`, `--plugin`, `--connect`, `--listen`, `--no-timestamp`.
6. Multicast scouting: `--no-multicast-scouting` disables it. Otherwise, if the file didn't set
   `scouting/multicast/enabled`, it's set to `true`.
7. Apply `--adminspace-permissions`.
8. Apply each non-empty `--cfg KEY:VALUE` in order. Errors are logged as warnings and don't stop startup.

!!! warning "The admin space is always on in zenohd"
    `adminspace.enabled: false` in your file has no effect under `zenohd`. Step 4 overrides it. Use
    `--adminspace-permissions none`, `adminspace/permissions`, or [access control](../security/access-control.md)
    on `@/**` to lock it down.

!!! tip "`--cfg` happens last"
    Because `--cfg` runs last, `--cfg 'adminspace/enabled:false'` is the one way to really disable the admin
    space in `zenohd`.

## Examples

```bash
# Router with defaults: listens on tcp/[::]:7447, multicast scouting on, admin space read-only.
zenohd

# Router from a file, with REST on port 8000 and read-write admin space.
zenohd -c router.json5 --rest-http-port 8000 --adminspace-permissions rw

# Two routers forming a backbone.
zenohd -l tcp/0.0.0.0:7447 -e tcp/10.0.0.2:7447
zenohd -l tcp/0.0.0.0:7447 -e tcp/10.0.0.1:7447

# Load the storage manager with an in-memory storage, without a config file.
zenohd -P storage_manager \
  --cfg 'plugins/storage_manager/storages/demo:{key_expr:"demo/example/**",volume:"memory"}'
```

## Logging

`zenohd` reads `RUST_LOG` through `tracing_subscriber::EnvFilter`. When `RUST_LOG` isn't set, the filter is
`z=info`, which matches every target whose name starts with `z` (all `zenoh*` crates). Log lines include
thread IDs, thread names, level and target.

Useful filters:

```bash
RUST_LOG=zenoh=debug zenohd                          # everything at debug
RUST_LOG=zenoh::net::routing::interceptor=trace zenohd  # see each message ACL/downsampling drops
RUST_LOG=zenoh_transport=debug,zenoh=info zenohd     # transport/link establishment details
```

## Running as a service

The repo ships `zenohd/.service/zenohd.service` (systemd) and `zenohd/.service/zenohd.json5`. The unit runs
`/usr/bin/zenohd -c /etc/zenohd/zenohd.json5` as user `zenohd`, with `RUST_LOG=info` and
`ZENOH_HOME=/var/zenohd`, stops with `SIGINT`, and restarts on failure.

## Features compiled in

See [Cargo feature flags](feature-flags.md#what-zenohd-is-built-with). In short, a stock build has
`unstable` and plugin support, and lacks SHM, stats and io_uring.

## Sources

- `zenohd/src/main.rs`
- `zenohd/Cargo.toml`
- `zenohd/.service/`
