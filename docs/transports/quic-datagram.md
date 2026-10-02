# QUIC datagram

`quic/...?rel=0` opens a QUIC connection and carries Zenoh batches in **unreliable QUIC DATAGRAM frames**
(RFC 9221). The traffic is encrypted like QUIC, but nothing is retransmitted, which suits UDP-like data
that must still be encrypted.

| Property | Value |
|---|---|
| Feature | `transport_quic_datagram` (default) |
| Locator | `quic/<host>:<port>?rel=0` (same `quic` prefix; `rel=0` selects this link) |
| Reliable | no |
| Max batch | QUIC `max_datagram_size` **when the link opens**, cached for the link's lifetime. With quinn's default `initial_mtu` (1200) that's about 1.2 kB |
| Encryption | TLS 1.3 |

## How the dispatch works

`io/zenoh-link/src/lib.rs` picks the implementation from the locator's reliability:

- Both QUIC features on (the default): `rel=0` → datagram link, otherwise → stream link.
- Only `transport_quic`: `rel=0` fails with `Cannot use unreliable QUIC without enabling the transport_quic_datagram feature`.
- Only `transport_quic_datagram`: `quic/` locators must be unreliable.

## Things to watch

- **Small MTU**: datagrams are bounded by the path MTU. Most Zenoh messages bigger than about 1 kB are
  **fragmented**, and losing one fragment loses the whole message. Raise `#initial_mtu=1400` (or more on
  networks you control) to cut fragmentation.
- The MTU is read once, when the link opens. Later path-MTU discovery doesn't increase it for that link.
- To send reliable and best-effort traffic over **one** connection, use [`mixed_rel`](quic.md#mixed_rel) on
  a reliable `quic/` locator instead of a separate `rel=0` link.

The TLS/certificate settings are the same as for [QUIC](quic.md) and [TLS](tls.md).

## Sources

- `io/zenoh-links/zenoh-link-quic_datagram/src/`
- `io/zenoh-link/src/lib.rs`
