# QUIC

`quic/` runs Zenoh over QUIC (the `quinn` crate, TLS 1.3). It's encrypted like [TLS](tls.md) and can also
map Zenoh priorities onto separate QUIC streams, which avoids head-of-line blocking between priorities.

| Property | Value |
|---|---|
| Feature | `transport_quic` (default). Best-effort mode needs `transport_quic_datagram` (default) |
| Reliable | yes. `?rel=0` switches to [QUIC datagrams](quic-datagram.md) |
| Max batch | 65535 |
| Encryption | TLS 1.3 (always) |
| Certificates | Same settings as TLS (`transport/link/tls/*` and the per-endpoint `*_file`/`*_raw`/`*_base64` keys) |
| io_uring | not supported |

## Locator

```text
quic/router.example.com:7447
quic/0.0.0.0:7447?multistream=1;mixed_rel=1#initial_mtu=1400
```

## Metadata (`?`) options

These are part of the locator, so they're advertised to peers.

### `multistream`

| Value | Meaning |
|---|---|
| `auto` (default) | Use per-priority streams if the other side supports them |
| `1` | Require per-priority streams |
| `0` | One stream for everything (as before) |

With multistream, the control priority uses the bidirectional stream and each of the other 7 priorities gets
its own **unidirectional QUIC stream**. Each stream's QUIC priority is the negation of the Zenoh priority,
so `real_time` is scheduled ahead of `background`. A lost packet on the `background` stream doesn't hold up
`real_time` data.

### `mixed_rel`

| Value | Meaning |
|---|---|
| `0` (default) | Reliable streams only |
| `1` | Require mixed reliability |
| `auto` | Use it if the other side supports it |

With mixed reliability, **one** QUIC connection carries two Zenoh links: the reliable stream link and a
best-effort **datagram** link. Best-effort messages (`Reliability::BestEffort`) use QUIC datagrams and
avoid retransmission. In the transport this is a `NewLink::MixedReliability { reliable, best_effort }`.

### Negotiation

The two options are negotiated with QUIC **ALPN**. Each side offers protocols in order of preference:

| ALPN | Multistream | Mixed reliability |
|---|---|---|
| `zenoh-ms-mr` | ✅ | ✅ |
| `zenoh-ms` | ✅ | ❌ |
| `zenoh-mr` | ❌ | ✅ |
| `zenoh` | ❌ | ❌ |
| `hq-29` | ❌ | ❌ (legacy, for older Zenoh) |

`1` offers only the variants with the feature, `0` only the variants without it, and `auto` offers both
(preferring the feature). If one side requires a feature (`1`) and the other disables it (`0`), they share no
ALPN and the connection fails.

## Config (`#`) options

| Key | Meaning |
|---|---|
| `initial_mtu` | QUIC initial path MTU (u16). quinn's default is 1200 |
| `mtu_discovery_interval_secs` | Interval for path-MTU discovery |
| `iface`, `bind`, `dscp` | As for TCP/UDP. `iface` and `bind` together are rejected |
| TLS keys | `root_ca_certificate_*`, `listen_*`, `connect_*`, `enable_mtls`, `verify_name_on_connect`, `close_link_on_expiration` (see [TLS](tls.md)) |

## Good at / limits

- ✅ No head-of-line blocking across priorities (multistream), fast loss recovery, built-in congestion
  control, connection setup on UDP.
- ✅ Encrypted, with certificate CNs usable as ACL identities (`cert_common_names`).
- ✅ Reliable and best-effort traffic on one connection (`mixed_rel`).
- ⚠️ Needs certificates (or use [`udp/...?rel=1`](udp.md#reliable-udp) for unencrypted QUIC).
- ⚠️ UDP may be blocked by firewalls. Raise UDP buffers for high throughput.
- ⚠️ Uses more CPU than TCP (userspace stack, encryption).

## Sources

- `io/zenoh-links/zenoh-link-quic/src/`
- `io/zenoh-link-commons/src/quic/` (`unicast.rs`: `MultiStreamConfig`, `MixedRelConfig`, ALPN; `utils.rs`: MTU options)
- `io/zenoh-link/src/lib.rs` (QUIC vs QUIC-datagram dispatch)
