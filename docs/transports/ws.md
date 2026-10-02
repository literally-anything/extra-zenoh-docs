# WebSocket

`ws/` carries Zenoh batches as binary WebSocket messages (`tokio-tungstenite`). Use it where only HTTP-like
traffic gets through, or as a bridge for web clients.

| Property | Value |
|---|---|
| Feature | `transport_ws` (default) |
| Reliable | yes |
| Byte stream | no. Each batch is one WebSocket binary message |
| Max batch | 65535 |
| Encryption | **none**. The URL is always `ws://`; `wss` isn't supported |
| io_uring | not supported |

## Locator

```text
ws/0.0.0.0:8080
ws/gateway.example.com:8080
```

The link dials `ws://<resolved-address>:<port>` with no path. Only binary frames carry data.

## Good at / limits

- ✅ Passes through HTTP proxies and load balancers that understand WebSocket upgrades.
- ⚠️ **No TLS of its own.** For encryption over the internet, put a TLS-terminating reverse proxy in front,
  or use `tls/` or `quic/`.
- ⚠️ Slightly more framing overhead than TCP.

!!! note "Browsers and zenoh-ts"
    The TypeScript binding (zenoh-ts) doesn't speak the Zenoh protocol over `ws/`. It talks to the
    **remote-api plugin** inside `zenohd` over its own WebSocket API. See [TypeScript](../api/languages/typescript.md)
    and [Ecosystem plugins](../plugins/ecosystem.md).

## Sources

- `io/zenoh-links/zenoh-link-ws/src/`
