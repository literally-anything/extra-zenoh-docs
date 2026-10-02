# UDP

`udp/` gives you three different things depending on the address and metadata:

| Locator | Becomes | Reliable |
|---|---|---|
| `udp/<unicast-ip>:<port>` | Plain UDP datagrams | no |
| `udp/<unicast-ip>:<port>?rel=1` | **QUIC without TLS** over UDP | yes |
| `udp/<multicast-ip>:<port>` | A multicast transport shared by every group member | no |

| Property | Value |
|---|---|
| Feature | `transport_udp` (default) |
| Max batch | 65487 on Linux/Windows (65535 − 8 UDP − 40 IPv6 header), 9216 on macOS, 8192 elsewhere |
| io_uring | connected unicast sockets only |

## Best-effort unicast

```text
udp/192.168.1.10:7447
```

- Each batch is one datagram. Nothing is retransmitted: a lost datagram loses every message in it, and a
  lost fragment loses the whole fragmented message.
- Listeners use one **unconnected** socket shared by all remotes and demultiplex by source address. The
  dialling side uses a **connected** socket. Only connected sockets can use io_uring.
- Good for high-rate sensor data where a late sample is worthless.

!!! tip "Raise UDP buffers for throughput"
    The link source recommends raising host UDP buffers to about 4 MiB for high-throughput use:
    ```bash
    sysctl -w net.core.rmem_max=4194304
    sysctl -w net.core.rmem_default=4194304
    ```

## Reliable UDP

```text
udp/192.168.1.10:7447?rel=1
```

With `rel=1` on a unicast address, the UDP link runs **QUIC with security turned off**
(`QuicClientBuilder::security(false)`). You get QUIC's reliability, congestion control and stream
multiplexing without certificates or encryption. The QUIC metadata options `multistream` and `mixed_rel`
work here too (with `mixed_rel`, best-effort traffic travels as unencrypted QUIC datagrams on the same connection).

Use it when you want QUIC's behaviour on lossy networks without managing certificates, on networks you trust.

## Multicast

```text
udp/224.0.0.225:7447#iface=eth0;ttl=4;join=224.0.0.226|224.0.0.227
```

Listening or connecting on a multicast address creates a [multicast transport](transport-layer.md#multicast-transports).

| Config key | Meaning |
|---|---|
| `iface` | Interface name or local IP to send from. Without it, the first non-loopback multicast-capable address of the matching IP family is used |
| `bind` | Explicit local `ip:port` for the sending socket |
| `ttl` | Multicast TTL (IPv4 only. On IPv6 a warning is logged and it's ignored: no hop-limit support yet) |
| `join` | Extra multicast groups to join for receiving, separated by `|` |
| `dscp` | ToS byte for sent datagrams |

Rules:

- Every member must agree on `batch_size`, QoS, compression and sequence-number resolution (no handshake).
- Interceptors (ACL, downsampling, …) don't apply.
- Don't use the scouting address (`224.0.0.224:7446`) for data.

## Unicast config keys

| Key | Meaning |
|---|---|
| `iface` | `SO_BINDTODEVICE` (Linux/Android) |
| `bind` | Local `ip:port` for the dialling socket |
| `dscp` | ToS byte |

## Sources

- `io/zenoh-links/zenoh-link-udp/src/` (`lib.rs`, `unicast.rs`, `multicast.rs`, `reliability.rs`)
