# Tuning

Starting points for common goals. Every setting here is described in the [transport layer](transport-layer.md)
page and the [config reference](../configuration/reference.md). Measure before and after: Zenoh ships
`z_pub_thr`/`z_sub_thr` and `z_ping`/`z_pong` in `examples/` for this.

## Low latency

```json5
transport: {
  unicast: {
    // Optional, and only for 1-to-1 small-message traffic between two nodes:
    // lowlatency: true, qos: { enabled: false },
  },
  link: {
    tx: {
      queue: {
        batching: { enabled: true, time_limit: 1 },   // adaptive batching costs nothing when idle
      },
    },
  },
},
```

- Mark latency-critical publications **express** (`express(true)`) so they skip batching. You can do this
  without code changes using [`qos/publication`](../configuration/qos-overwrite.md).
- Give them a high **priority** (`real_time`/`interactive_high`) so they're queued ahead of bulk data.
- Avoid head-of-line blocking: put critical priorities on their own link (`?prio=1-2`) with
  [multilink](transport-layer.md#multilink), or use [QUIC multistream](quic.md#multistream).
- On the same host, use [shared memory](../shm/index.md) for payloads larger than a few KB.
- On Linux, try [io_uring](io-uring.md) for TCP/Unix-socket RX.

## High throughput

```json5
transport: {
  link: {
    tx: {
      batch_size: 65535,
      queue: {
        size: { data: 16, data_low: 16, background: 16 },  // max 16 batches per queue
        batching: { enabled: true, time_limit: 1 },
      },
    },
    rx: {
      buffer_size: 16777216,     // 16 MiB RX buffer; also sizes the io_uring buffer ring
    },
    tcp: { so_sndbuf: 8388608, so_rcvbuf: 8388608 },
  },
},
```

- Raise OS socket limits to match (`net.core.rmem_max`, `net.core.wmem_max`).
- For many small messages, batching does most of the work. Don't mark them express.
- For large payloads on one host, use [SHM](../shm/index.md) (the transport optimisation copies payloads ≥ 3 KB into SHM automatically).
- Consider [compression](transport-layer.md#compression) on slow links with compressible data.

## Constrained devices and links

```json5
transport: {
  link: {
    tx: {
      batch_size: 1500,
      queue: {
        size: { control: 1, real_time: 1, interactive_high: 1, interactive_low: 1,
                data_high: 1, data: 2, data_low: 1, background: 1 },
        allocation: { mode: "lazy" },
      },
    },
    rx: {
      buffer_size: 4096,
      max_message_size: 1048576,   // 1 MiB instead of 1 GiB
    },
  },
  shared_memory: { enabled: false },
},
```

- Use `client` mode so the device keeps one connection.
- Limit what reaches the device with [downsampling](../configuration/downsampling.md) and
  [low-pass filters](../configuration/low-pass-filter.md) on the router side (egress towards the device).

## Lossy or high-latency networks

- Use `quic/` (with `multistream`) or `udp/...?rel=1` instead of TCP.
- Increase `transport/link/tx/lease` (for example 30000 ms) so short outages don't tear sessions down.
- Send time-critical but loss-tolerant data best-effort (`reliability: best_effort`, unstable) over
  `mixed_rel` or a separate `rel=0` link.
- Keep `congestion_control: drop` on telemetry so a slow link can't block publishers.

## Startup time

- `open/return_conditions/connect_scouted: false` and `declares: false` make `zenoh::open` return sooner,
  at the cost of possibly missing the first publications.
- `scouting/delay` (500 ms) bounds how long peers wait for scouting during open.
- `transport/shared_memory/mode: "init"` moves SHM setup to open time instead of the first message.

## Sources

- `DEFAULT_CONFIG.json5`, `commons/zenoh-config/src/defaults.rs`
- `examples/examples/z_pub_thr.rs`, `z_sub_thr.rs`, `z_ping.rs`, `z_pong.rs`
