# Transports

Zenoh separates **links** (one connection over one protocol: a TCP socket, a QUIC connection, a serial
port) from **transports** (the Zenoh session between two nodes, which may use several links). This section
covers both.

- [Transport layer](transport-layer.md): what sits above the links. Batching, priority queues,
  congestion control, fragmentation, leases, multilink, low-latency mode, compression, multicast transports.
- One page per link protocol (below).
- [Tuning](tuning.md): settings for throughput, latency and constrained devices.
- [io_uring](io-uring.md): the optional Linux RX path.

## Comparison

All values are from the link crates in `io/zenoh-links/`.

| Locator | Cargo feature (default?) | Reliable by default | Byte stream | Max batch / MTU | Encryption | Multicast | io_uring RX | Platforms |
|---|---|---|---|---|---|---|---|---|
| [`tcp/`](tcp.md) | `transport_tcp` ✅ | yes | yes | ≈65.4k ([aligned to MSS](tcp.md#effective-mtu)) | no | no | ✅ | all |
| [`udp/`](udp.md) | `transport_udp` ✅ | **no** | no | 65487 (Linux/Windows), 9216 (macOS), 8192 (other) | no | **yes** | ✅ (connected only) | all |
| [`udp/…?rel=1`](udp.md#reliable-udp) | `transport_udp` ✅ | yes (QUIC without TLS) | no | 65535 | no | no | ❌ | all |
| [`tls/`](tls.md) | `transport_tls` ✅ | yes | yes | ≈65.4k (same MSS alignment as TCP) | TLS 1.2/1.3 (1.3 with mTLS) | no | ❌ | all |
| [`quic/`](quic.md) | `transport_quic` ✅ | yes | no (QUIC streams) | 65535 | QUIC/TLS 1.3 | no | ❌ | all |
| [`quic/…?rel=0`](quic-datagram.md) | `transport_quic_datagram` ✅ | **no** | no | QUIC `max_datagram_size` when the link opens (≈1.2 kB with default `initial_mtu`) | QUIC/TLS 1.3 | no | ❌ | all |
| [`ws/`](ws.md) | `transport_ws` ✅ | yes | no (WS messages) | 65535 | no (no `wss`) | no | ❌ | all |
| [`unixsock-stream/`](unixsock-stream.md) | `transport_unixsock-stream` ✅ | yes | yes | 65535 | n/a (local) | no | ✅ | Unix |
| [`unixpipe/`](unixpipe.md) | `transport_unixpipe` ❌ | yes | yes | 65535 | n/a (local) | no | ✅ | Unix |
| [`serial/`](serial.md) | `transport_serial` ❌ | **no** | no (COBS frames) | **1500** | no | no | ❌ | all |
| [`vsock/`](vsock.md) | `transport_vsock` ❌ | yes | yes | 65535 | n/a (VM-host) | no | ✅ | Linux |

*Max batch / MTU* is the largest Zenoh batch the link carries in one unit. Bigger messages are
**fragmented** by the transport and reassembled at the other end, up to `transport/link/rx/max_message_size`
(1 GiB by default). The effective batch size is the smallest of `transport/link/tx/batch_size`, the remote's
value and the link MTU.

*Reliable* is what the link reports unless the locator's `rel` metadata overrides it. Zenoh uses it to place
reliable and best-effort messages when there are several links (see [Multilink](transport-layer.md#multilink)).

## Which one should I use?

| Situation | Recommended | Why |
|---|---|---|
| General LAN/WAN, default | `tcp` | Reliable, simple, supports io_uring |
| Untrusted networks | `tls` or `quic` | Encryption, server and optional client authentication; certificate CNs usable in [ACL](../security/access-control.md) |
| Lossy or long-distance links, many priorities | `quic` with `multistream` | No head-of-line blocking between priorities, faster recovery from loss, connection migration |
| Real-time data where late = useless | `udp` (`rel=0`), or `quic` with `mixed_rel` | No retransmission delays |
| One-to-many on a LAN | `udp/<multicast-group>` | One send reaches all peers in the group |
| Browser or HTTP-only network paths | `ws` | Passes through HTTP infrastructure; used by zenoh-ts through the remote-api plugin |
| Same host | `unixsock-stream` (or [shared memory](../shm/index.md) for payloads) | No TCP/IP overhead |
| Microcontrollers over UART | `serial` | Works with zenoh-pico; framing and CRC built in |
| VM to host or between VMs | `vsock` | No virtual network needed |

## Restricting protocols

`transport/link/protocols` is a whitelist for both listening and connecting:

```json5
transport: { link: { protocols: ["tls", "quic"] } }
```

With this setting, `tcp/...` endpoints (including the default listener) are rejected.

## Sources

- `io/zenoh-links/*/src/lib.rs` (MTU, reliability, locator prefixes)
- `io/zenoh-link/src/lib.rs` (protocol dispatch, QUIC vs QUIC-datagram selection)
- `io/zenoh-link-commons/src/unicast.rs` (`get_fd`, mixed reliability)
