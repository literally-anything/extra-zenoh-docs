# Shared memory (zero-copy)

With shared memory (SHM), processes **on the same host** exchange large payloads without copying them
through sockets. The publisher writes into a shared-memory buffer, and only a small descriptor travels
over the Zenoh link. The subscriber maps the same memory and reads the data in place.

## Requirements

| Requirement | Detail |
|---|---|
| Cargo feature | `shared-memory` on **both** sides. The `zenoh::shm` API also needs `unstable` |
| Config | `transport/shared_memory/enabled: true` (the default) on **both** sides |
| Same host | Both processes must see the same POSIX shared memory (`/dev/shm` on Linux). In containers, share the IPC namespace (`--ipc=host` or a shared `/dev/shm`) |
| Bindings | See the [API matrix](../api/index.md) for SHM support per language |

`zenohd` isn't built with `shared-memory` by default. Rebuild with `--features shared-memory`.

## How it works

```mermaid
sequenceDiagram
    participant P as Publisher process
    participant SHM as /dev/shm segment
    participant S as Subscriber process
    P->>SHM: provider.alloc(n) → ZShmMut, write data
    P->>S: Zenoh message carrying an SHM descriptor (segment, chunk, len)
    S->>SHM: map segment (once), read chunk in place
    Note over S: payload is a ZShm; zero copies
    S-->>SHM: buffer released when dropped (refcount in SHM)
```

1. **Probing.** While the transport is set up, each side checks that it can read a challenge written to the
   other side's shared memory. Only then is SHM turned on for the transport (`Transport::is_shm()`, and
   `"shm": true` in the admin space). Otherwise data goes over the network as usual, with no error.
2. **Explicit SHM.** The application allocates from an `ShmProvider` and publishes the buffer
   (see [SHM API](api.md)). Over an SHM-capable transport, only a descriptor is sent.
3. **Implicit SHM (transport optimisation).** When `transport_optimization` is on (the default), regular
   payloads ≥ `message_size_threshold` (3072 bytes) of the selected message types are **copied once** into
   an internal SHM pool (16 MiB) and sent as SHM. This saves copies on the receiving side without changing
   application code.
4. **Fallback.** If the remote can't use SHM (different host, feature off), an SHM payload is copied into a
   regular buffer and sent over the network. Applications don't need to handle this: `ZBytes` works either way.

## Safety mechanisms

- **Reference counting** in shared memory tracks how many processes hold a chunk. Memory is reclaimed by
  the provider's garbage collection once nobody references it.
- **Watchdog**: processes holding SHM buffers confirm them periodically (every **50 ms**, `WatchdogConfirmator`),
  and a validator checks every **100 ms** (`WatchdogValidator`). Buffers held by a process that stops
  confirming, because it crashed for example, are invalidated and reclaimed.
- **Orphaned segment cleanup** (Linux): POSIX segments outlive a crashed creator. Zenoh scans `/dev/shm` for
  segments no process uses and removes them, on the first SHM segment creation and at normal process exit.
  You can also trigger it yourself with `zenoh::shm::cleanup_orphaned_shm_segments()`.
- `RLIMIT_NOFILE` is raised to the hard limit the first time segments are created or opened, because each
  segment uses a file descriptor.

!!! warning "Known race under heavy CPU load"
    Upstream issue [#2628](https://github.com/eclipse-zenoh/zenoh/issues/2628): with implicit SHM, a
    query or reply buffer can be invalidated by the watchdog while it's still in transit when the system is
    under heavy CPU load. The workaround is to remove `query` and `reply` from
    `transport/shared_memory/transport_optimization/messages`. See [SHM configuration](config.md).

## When SHM helps

- ✅ Large payloads (images, point clouds, tensors) between processes on one machine.
- ✅ Fan-out to many local subscribers: the data is written once and every subscriber maps it.
- ⚠️ Small payloads (a few KB or less) gain little. That's why the implicit threshold is 3 KB.
- ⚠️ Across hosts it does nothing (it falls back to the network).

## Sources

- `commons/zenoh-shm/src/` (`api/`, `watchdog/`, `posix_shm/cleanup.rs`, `init.rs`)
- `io/zenoh-transport/src/unicast/establishment/ext/shm/` (probing challenge)
- `commons/zenoh-config/src/lib.rs` (`ShmConf`), `DEFAULT_CONFIG.json5`
- `examples/examples/z_pub_shm.rs`, `z_sub_shm.rs`, `z_alloc_shm.rs`
