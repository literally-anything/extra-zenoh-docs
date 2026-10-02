# Endpoints & locators

Endpoints tell Zenoh where to **listen** (`listen/endpoints`) and where to **connect**
(`connect/endpoints`). Scouting uses them too, to advertise and dial discovered nodes.

## Grammar

```
<protocol>/<address>[?<metadata>][#<config>]
```

| Part | Separator | Format | Travels to the remote? |
|---|---|---|---|
| protocol | — | `tcp`, `udp`, `tls`, `quic`, `ws`, `unixsock-stream`, `unixpipe`, `serial`, `vsock` | yes |
| address | `/` | Protocol-specific (`host:port`, path, device, `cid:port`) | yes |
| metadata | `?` | `key=value;key=value` | **yes**: it's part of the *locator* that gets advertised in scouting |
| config | `#` | `key=value;key=value` | **no**: local socket options only |

A **locator** is the `protocol/address?metadata` part. An **endpoint** is a locator plus local config.
`Config` key-value lists use `;` between entries, `=` between key and value, and `|` between multiple values.
The canonical form sorts keys alphabetically.

Examples:

```text
tcp/192.168.1.10:7447
tcp/[::]:7447
tcp/localhost:7447?prio=1-3;rel=1#iface=eth0;so_sndbuf=1048576
udp/224.0.0.225:7447#iface=eth0;ttl=4
tls/router.example.com:7447#root_ca_certificate_file=/etc/zenoh/ca.pem
quic/0.0.0.0:7447?multistream=1
unixsock-stream//tmp/zenoh.sock
serial//dev/ttyUSB0#baudrate=115200
vsock/VMADDR_CID_ANY:7447
```

## Metadata

Metadata is part of the locator, so it's advertised to other nodes and helps choose which link carries a message.

| Key | Values | Meaning | Supported by |
|---|---|---|---|
| `prio` | `a-b` (inclusive, 0–7; 0 is `control`) or a single value | This link carries only messages with priority in `a..=b`. Pair several endpoints to the same peer with different ranges to build **priority-dedicated links**. | All unicast links |
| `rel` | `0` (best effort) or `1` (reliable) | Overrides the link's reported reliability, so best-effort traffic can be steered to one link and reliable traffic to another. A link's default comes from its protocol (TCP reliable, UDP best effort, …). | All unicast links |
| `multistream` | `auto` (default), `0`, `1` | QUIC only: one QUIC stream per priority. See [QUIC](../transports/quic.md#multistream) | `quic` |
| `mixed_rel` | `0` (default), `1`, `auto` | QUIC only: send best-effort traffic as QUIC datagrams on the same connection. See [QUIC](../transports/quic.md#mixed_rel) | `quic` |

!!! example "Separate links per priority and reliability"
    ```json5
    connect: {
      endpoints: [
        "tcp/10.0.0.1:7447?prio=1-2;rel=1",   // real-time and interactive-high, reliable
        "tcp/10.0.0.1:7448?prio=3-7;rel=1",   // everything else, reliable
        "udp/10.0.0.1:7449?rel=0",            // best-effort traffic
      ],
    },
    transport: { unicast: { max_links: 3 } },  // needs transport_multilink
    ```
    The remote must listen on matching endpoints, and both sides need `max_links` > 1. See
    [Multilink](../transports/transport-layer.md#multilink).

## Config (`#`) options

These options never leave the node. Each [transport page](../transports/index.md) has the full list. The
common ones are:

| Key | Applies to | Meaning |
|---|---|---|
| `iface` | TCP, TLS, UDP (unicast), QUIC | **Linux/Android**: bind the socket to a network device (`SO_BINDTODEVICE`). On other OSes it logs a warning and does nothing. On a listener bound to `0.0.0.0`/`[::]`, it also limits which addresses get advertised. |
| `bind` | TCP, TLS, UDP (unicast and multicast), QUIC | Local `ip:port` for outgoing connections. The IP family must match the destination's. **Can't be combined with `iface`.** |
| `so_sndbuf`, `so_rcvbuf` | TCP, TLS | Kernel socket buffer sizes in bytes. Override `transport/link/tcp/*` and `transport/link/tls/*`. |
| `dscp` | TCP, TLS, UDP, QUIC | Value written to `IP_TOS` (IPv4) or `IPV6_TCLASS` (IPv6). Decimal or `0x` hex; several values can be OR-ed with `|` (`0x04|0x10`). |
| `exit_on_failure` | connect and listen | Per-endpoint override of `connect/exit_on_failure` / `listen/exit_on_failure` |
| `retry_period_init_ms`, `retry_period_max_ms`, `retry_period_increase_factor` | connect and listen | Per-endpoint override of the `retry` block |

!!! warning "`dscp` sets the whole ToS byte"
    The value goes to `IP_TOS`/`IPV6_TCLASS` unchanged. That byte holds the 6-bit DSCP **shifted left by 2**
    plus 2 ECN bits. For DSCP class EF (46), write `dscp=0xB8` (46 << 2 = 184), not `dscp=46`.

!!! note "Binary prefix quirk"
    The DSCP parser accepts `0x`/`0X` for hex. For binary it looks for `0xb` or `0B`, so lowercase `0b101`
    isn't understood. Use hex or decimal.

## Endpoint groups

An entry in `connect/endpoints` can also be a **group**:

```json5
connect: {
  endpoints: [
    { strategy: "allOf", locators: ["tcp/10.0.0.1:7447?rel=1", "udp/10.0.0.1:7447?rel=0"] },
    "tcp/10.0.0.2:7447",
  ],
}
```

- `allOf`: open links to **all** locators in the group. This is what a **client** uses to get several links
  (for example reliable and best-effort) to its one gateway.
- `oneOf`: accepted by the parser but **not implemented**. It logs
  `locator groups with strategy=oneOf are not implemented yet; falling back to current allOf behavior`.

For peers and routers a group behaves the same as listing its locators one by one.

## How connect/listen behave per mode

From `zenoh/src/net/runtime/orchestrator.rs`:

**Listening.** All `listen/endpoints` for the current mode are bound at startup:

- `listen/timeout_ms = 0` (the default): each listener is tried once. If it fails and `exit_on_failure` is
  true (the default), session open fails. Otherwise the listener is skipped.
- Non-zero timeout with `exit_on_failure: true`: the listener is retried with the `listen/retry` backoff,
  and session open waits, up to the global `listen/timeout_ms` (`-1` = forever).
- Non-zero timeout with `exit_on_failure: false`: the listener is retried **in the background** and the
  session opens straight away.

**Clients.** A client walks `connect/endpoints` **in order** and stops at the first entry (endpoint or
group) that connects. If no endpoints are configured, it scouts with UDP multicast for up to
`scouting/timeout` (3 s) and connects to the first node whose mode matches
`scouting/multicast/autoconnect/client`. If the connection drops, the client goes through its configured
endpoints again using the retry backoff.

**Peers and routers.** They try **every** endpoint in `connect/endpoints`. With the defaults
(`connect/timeout_ms = -1`, `exit_on_failure = false`) each endpoint gets its own background task that
retries forever. When a link to a configured endpoint closes, that endpoint is retried in the background.

### Timeouts and retries

| Key | Meaning |
|---|---|
| `connect/timeout_ms`, `listen/timeout_ms` | Total time allowed for the connect/listen phase. `0` = try once without retry. `-1` = no limit. |
| `*/retry/period_init_ms` | First delay between attempts (default 1000). `< 0` = wait forever; `0` = no delay. |
| `*/retry/period_max_ms` | Upper bound on the delay (default 4000). `≤ 0` = no upper bound. |
| `*/retry/period_increase_factor` | Multiplier applied to the delay after each attempt (default 2). |
| `*/exit_on_failure` | When the timeout runs out, fail session open (`true`) or keep trying in the background (`false`). |

Defaults: `connect/timeout_ms` is router `-1`, peer `-1`, client `0`. `connect/exit_on_failure` is router
`false`, peer `false`, client `true`. `listen/timeout_ms` is `0` and `listen/exit_on_failure` is `true`.

## Listening on wildcard addresses

When you listen on `0.0.0.0` or `[::]`, Zenoh lists every address of the host (or of `iface`, if set) as
a locator. Two lists exist:

- **All locators**: used in scouting `Hello` replies sent to a **loopback** scouter.
- **Locators without loopback**: used in replies to remote scouters, and logged at startup as
  `Zenoh can be reached at: ...`.

Port `0` asks the OS for a free port. Peers use this by default (`tcp/[::]:0`), and the real port is what gets advertised.

## Sources

- `commons/zenoh-protocol/src/core/endpoint.rs` (grammar, metadata keys, `LocatorsStrategy`)
- `commons/zenoh-config/src/connection_retry.rs`
- `io/zenoh-link-commons/src/lib.rs`, `tcp.rs`, `dscp.rs`, `listener.rs`, `quic/`
- `commons/zenoh-util/src/net/mod.rs` (`set_bind_to_device_*`)
- `zenoh/src/net/runtime/orchestrator.rs`
