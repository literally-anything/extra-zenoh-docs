# TCP

`tcp/` is Zenoh's default link. Routers listen on `tcp/[::]:7447` and peers on `tcp/[::]:0` unless you
configure something else.

| Property | Value |
|---|---|
| Feature | `transport_tcp` (default) |
| Reliable | yes (override with `?rel=0`) |
| Byte stream | yes. Each batch gets a 2-byte length prefix |
| Max batch | 65535 bytes (limited by the 16-bit length prefix, not by TCP) |
| Encryption | none. Use [TLS](tls.md) or [QUIC](quic.md) |
| io_uring | supported |
| Listener backlog | 1024 |

## Locator

```text
tcp/<host-or-ip>:<port>[?<metadata>][#<config>]
tcp/192.168.1.10:7447
tcp/[fe80::1%eth0]:7447
tcp/router.example.com:7447
```

Hostnames are resolved with the system resolver. Multicast addresses are filtered out.

## Endpoint config (`#`)

| Key | Example | Meaning |
|---|---|---|
| `iface` | `iface=eth0` | Bind the socket to a device (`SO_BINDTODEVICE`, Linux/Android only). On a wildcard listener, only that interface's addresses are advertised |
| `bind` | `bind=192.168.1.5:0` | Local address for outgoing connections. Must be the same IP family as the target. Can't be combined with `iface` |
| `so_sndbuf` | `so_sndbuf=4194304` | Kernel send buffer (bytes) |
| `so_rcvbuf` | `so_rcvbuf=4194304` | Kernel receive buffer (bytes) |
| `dscp` | `dscp=0xB8` | IP ToS / traffic class byte (see [Endpoints](../configuration/endpoints.md#config-options)) |
| `exit_on_failure`, `retry_period_*` | | Per-endpoint retry (see [Endpoints](../configuration/endpoints.md#timeouts-and-retries)) |

## Global config

```json5
transport: {
  link: {
    tcp: {
      so_sndbuf: 4194304,   // applied to every tcp/ link unless overridden per endpoint
      so_rcvbuf: 4194304,
    },
  },
},
```

## Good at / limits

- ✅ Works everywhere, gets through most firewalls and NATs (outbound), simplest to operate.
- ✅ Reliable and ordered. With io_uring, the RX path is the most efficient of any link.
- ⚠️ **Head-of-line blocking**: one TCP connection carries every priority. A lost segment holds back all
  messages behind it, high priority ones included. Use [multilink](transport-layer.md#multilink) with
  per-priority links, or [QUIC multistream](quic.md#multistream), if that matters.
- ⚠️ Buffers: on high bandwidth-delay links, raise `so_sndbuf`/`so_rcvbuf` (and the OS limits
  `net.core.rmem_max`/`wmem_max`).

## Sources

- `io/zenoh-links/zenoh-link-tcp/src/` (`lib.rs`, `unicast.rs`, `utils.rs`)
- `io/zenoh-link-commons/src/tcp.rs`
