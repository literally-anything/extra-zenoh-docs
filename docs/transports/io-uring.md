# io_uring

Zenoh 1.10 can receive data on unicast links through **Linux io_uring** instead of tokio's epoll-based
reader. It's a build-time option. This page covers what it does, when it's used, and how to tell whether
it's active.

!!! info "Build-time only"
    There's **no configuration key** for io_uring. It's on if zenoh was built with the `uring` Cargo feature
    on a supported platform. If it can't start, Zenoh quietly uses the normal path.

## Enabling

```bash
cargo build --release -p zenohd --features zenoh/uring
```

The `zenoh/uring` feature turns on `zenoh-transport/uring`, which turns on `zenoh-link/uring` and the
`uring` feature of every link crate.

### Platforms

io_uring is compiled in only on **Linux** with one of these architectures:
`x86_64`, `aarch64`, `riscv64`, `loongarch64`, `powerpc64`. On other platforms the feature is accepted and
a warning is logged:

```
The `uring` feature is enabled, but io_uring is only supported with Linux on x86_64, aarch64,
riscv64, loongarch64 or powerpc64; falling back to tokio RX.
```

### Kernel requirements

The ring is created with `IORING_SETUP_SUBMIT_ALL`, `IORING_SETUP_DEFER_TASKRUN` and
`IORING_SETUP_SINGLE_ISSUER`, and receives with **multishot `RecvMulti`** into provided-buffer rings. Those
flags need a recent kernel (`DEFER_TASKRUN` arrived in Linux 6.1). On older kernels, or where io_uring is
blocked (many container seccomp profiles block it, as do `kernel.io_uring_disabled=2` and some hardened
kernels), creating the ring fails and you get:

```
io_uring reactor init failed, falling back to tokio RX: <error>
```

## What uses it

When the transport manager starts, it tries to create one io_uring **reader** (`zenoh_uring::Reader`). It
runs a dedicated reactor thread (a blocking task on Zenoh's RX runtime) with a 4096-entry ring.

For each new unicast link, the RX task uses io_uring if the reader exists **and** the link exposes a raw file
descriptor:

| Link | io_uring RX |
|---|---|
| `tcp/` | ✅ |
| `udp/` connected socket (the dialling side) | ✅ |
| `udp/` unconnected socket (the listening side) | ❌ (shared socket, demultiplexed by source address in user space) |
| `udp/...?rel=1` (QUIC) | ❌ |
| `unixsock-stream/` | ✅ |
| `unixpipe/` | ✅ |
| `vsock/` | ✅ |
| `tls/`, `quic/`, `ws/`, `serial/` | ❌ (no usable raw fd) |

Links without an fd use the normal tokio RX task, even with the feature on. TX always uses the normal path.

## Sizing

The buffer ring is sized from the transport settings when the reader is created:

- buffer size = `transport/link/tx/batch_size` + 2 bytes (the length prefix on streamed links)
- buffer count = `max(transport/link/rx/buffer_size / buffer size, 16)`

With the defaults (65535 / 65535) that's **16 buffers of 65537 bytes** (about 1 MiB). To give the kernel
more room for in-flight data, raise `transport/link/rx/buffer_size`, for example to 16 MiB → about 256 buffers.

Streamed links (TCP, Unix socket, pipe, vsock) are reassembled from fragments with
`setup_fragmented_read`, and datagram links use `setup_read`. The lease timer is reset on every completion,
just like the tokio path.

## Checking it's active

- **No warning** at startup means the reader was created.
- With `RUST_LOG=zenoh_uring=debug` you'll see reactor commands (`Cmd: StartRx(...)`) as links open.
- `strace -f -e io_uring_setup,io_uring_enter` on the process shows the syscalls.

## When to use it

- ✅ Linux hosts receiving a lot of traffic over TCP or local sockets (routers, high-rate subscribers),
  where per-read syscalls and wakeups cost CPU.
- ⚠️ No benefit for TLS/QUIC/WS links, which keep the tokio path.
- ⚠️ Check that your container runtime allows io_uring. Otherwise you silently get the old path.

## Sources

- `commons/zenoh-uring/` (`src/linux/api/reader/mod.rs`, `Cargo.toml` target gating)
- `io/zenoh-transport/src/uring.rs` (buffer sizing), `manager.rs` (initialisation and fallback)
- `io/zenoh-transport/src/unicast/universal/link.rs` (`rx_task`, `rx_task_uring`)
- `io/zenoh-link-commons/src/unicast.rs` (`get_fd`), each link's `get_fd`
- `zenoh/Cargo.toml`, `io/zenoh-link/Cargo.toml` (`uring` feature wiring)
