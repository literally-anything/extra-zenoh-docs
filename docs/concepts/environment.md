# Environment variables and thread pools

A handful of environment variables change how Zenoh behaves. Two of them, `ZENOH_RUNTIME` and the logging
filter, matter a lot in production and are hardly mentioned in the official docs.

## Summary

| Variable | Read by | Effect |
|---|---|---|
| `ZENOH_CONFIG` | `Config::from_env()` (Rust), `zc_config_from_env` (C) and the other bindings' `from_env` | Path of a config file to load. **`zenohd` doesn't read it**; use `-c` |
| `RUST_LOG` | `zenohd`, `zenoh::init_log_from_env_or()`, `zenoh::try_init_log_from_env()`, `zc_init_log_from_env_or()` (C) | `tracing` filter. `zenohd` falls back to `z=info` when it's unset |
| `ZENOH_RUNTIME` | Every Zenoh process, at the first use of a pool | Sizes the internal Tokio thread pools (below) |
| `ZENOH_HOME` | `zenoh::internal::zenoh_home()`. Only the filesystem and RocksDB storage backends call it | Base directory, `~/.zenoh` by default. **Nothing in the core or `zenohd` uses it at 1.10.1.** Plugin search paths hard-code `~/.zenoh/lib` and ignore it |
| `UHLC_MAX_DELTA_MS` | The `uhlc` crate (Zenoh's hybrid logical clock) | Largest allowed clock drift for incoming timestamps, in ms (default **500**). A non-numeric value **panics** the first time the HLC is used. See [timestamping](../configuration/session-behaviour.md#timestamping) |
| `ZENOH_BACKEND_FS_ROOT` | filesystem backend | Root directory for its volumes (default `$ZENOH_HOME/zenoh_backend_fs`) |
| `ZENOH_BACKEND_ROCKSDB_ROOT` | RocksDB backend | Root directory for its databases (default `$ZENOH_HOME/zenoh_backend_rocksdb`) |
| `Z_LOG_PAYLOAD` | DDS and ROS 2 bridge plugins | If set, log payloads at trace level |
| `ROS_DOMAIN_ID`, `ROS_LOCALHOST_ONLY`, `ROS_AUTOMATIC_DISCOVERY_RANGE`, `ROS_DISTRO`, `CYCLONEDDS_URI` | ROS 2 bridge (and DDS bridge for the first one and `CYCLONEDDS_URI`) | Usual ROS 2 / Cyclone DDS meaning |

!!! note "The systemd unit sets `ZENOH_HOME` anyway"
    `zenohd/.service/zenohd.service` sets `ZENOH_HOME=/var/zenohd`. At 1.10.1 that only affects storage
    backends that call `zenoh_home()`. Plugins are still searched in `~/.zenoh/lib` of the service user.

## `ZENOH_RUNTIME`: thread pools

Zenoh doesn't use your application's async runtime for its own work. It runs five **Tokio multi-thread
runtimes** of its own, each created on first use:

| Pool | Default worker threads | What runs on it |
|---|---|---|
| `app` | 1 | Session-level helpers: synchronous bridges used by `info()` (ZID lists), zenoh-ext advanced subscriber and publication-cache tasks, cancellation tokens, SHM interop |
| `acc` | 1 | **Acceptor**: listener accept loops of every link, the accepting side of unicast handshakes, and the `scout()` API |
| `tx` | 1 | Transmit tasks: pull batches from the priority queues and write them to links |
| `rx` | **2** | Receive tasks: read links, decode batches, run ingress interceptors and routing, and **call your subscriber, queryable and reply callbacks** |
| `net` | 1 | Runtime housekeeping: multicast scouting sockets, link-state tree computation, query and interest timeouts, admin space, transport teardown |

Every pool also gets `max_blocking_threads: 50` for `spawn_blocking`. Threads are named `<pool>-<n>`
(`rx-0`, `rx-1`, `acc-0`, …), which helps when profiling.

The value is a [RON](https://github.com/ron-rs/ron) struct with one optional entry per pool:

```bash
ZENOH_RUNTIME='(
  rx:  (worker_threads: 4),
  tx:  (worker_threads: 2, max_blocking_threads: 8),
  acc: (handover: app),
  net: (handover: app),
)'
```

| Field | Meaning |
|---|---|
| `worker_threads` | Async worker threads (at least 1) |
| `max_blocking_threads` | Cap on threads for blocking tasks (at least 1) |
| `handover` | Don't build this pool; run its tasks on another pool instead (`app`, `acc`, `tx`, `rx` or `net`) |

Things to know:

- It's read **once per process**, when the first pool is created. Changing it afterwards has no effect.
- A pool is only built when something first uses it. A router with no connections has no `rx`/`tx` threads
  yet.
- An invalid value **panics** the process at session open, for example
  `NoSuchStructField { expected: ["app", "acc", "tx", "rx", "net"], found: "bogus" }`.
- `handover` cuts the thread count for small devices. `(acc: (handover: app), net: (handover: app))` saves
  two threads.
- Your callbacks run on `rx` threads. A slow callback holds up every link that thread serves. Raise
  `rx.worker_threads`, or move the work out of the callback (use a channel handler). See
  [Handlers](../api/handlers.md).
- Zenoh calls `block_in_place` internally, which **panics** under Tokio's current-thread scheduler. If your
  application uses Tokio, use the multi-thread flavour (one worker is enough):
  `#[tokio::main(flavor = "multi_thread", worker_threads = 1)]`.
- Calling the Zenoh API from an `atexit` handler panics with *"The Thread Local Storage inside Tokio is
  destroyed"*. The one exception is `close()`, which detects this case and closes from a fresh thread (it
  doesn't work with Rust 1.85.0–1.85.1). It's still best to close sessions before the process exits.

Verified with `zenohd` 1.10.1. With no `ZENOH_RUNTIME` and one connection, the process had threads
`acc-*`, `app-0`, `net-0`, `rx-0`, `rx-1`, `tx-0`. With
`(rx: (worker_threads: 4), acc: (handover: app), app: (worker_threads: 3))` it had four `rx-*` threads, no
`acc-*` threads, and the accept loops ran on `app-*`.

### `transport/link/tx/threads`

This config key (default 1) sets how many TX tasks a transport uses. It's separate from the `tx` pool size;
see [Transport layer](../transports/transport-layer.md#tx-threads).

## Logging

`RUST_LOG` uses the `tracing_subscriber::EnvFilter` syntax. Library sessions don't log until you call
`zenoh::init_log_from_env_or("error")` (Rust) or `zc_init_log_from_env_or("error")` (C), or the binding's
equivalent. See [zenohd logging](zenohd.md#logging) for useful filters.

## Sources

- `commons/zenoh-runtime/src/lib.rs` (`ZRuntime`, `RuntimeParam`, defaults)
- `commons/zenoh-macros/src/zenoh_runtime_derive.rs` (RON parsing, thread names)
- `io/zenoh-transport/src/unicast/universal/link.rs`, `multicast/link.rs` (pool per task)
- `zenoh/src/api/config.rs` (`ZENOH_CONFIG`), `commons/zenoh-util/src/lib.rs` (`ZENOH_HOME`)
- `commons/zenoh-util/src/lib_search_dirs.rs`, `zenohd/.service/zenohd.service`
- `uhlc` 0.8.2 `src/lib.rs` (`UHLC_MAX_DELTA_MS`)
- `zenoh-backend-filesystem/src/lib.rs`, `zenoh-backend-rocksdb/src/lib.rs`,
  `zenoh-plugin-ros2dds/src/config.rs`, `zenoh-plugin-dds/src/lib.rs`
