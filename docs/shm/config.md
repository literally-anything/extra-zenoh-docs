# SHM configuration & tuning

```json5
transport: {
  shared_memory: {
    enabled: true,
    mode: "lazy",                       // or "init"
    transport_optimization: {
      enabled: true,
      pool_size: 16777216,              // 16 MiB
      message_size_threshold: 3072,     // bytes
      messages: ["put", "query", "reply"],
    },
  },
},
```

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | Advertise SHM support while transports are set up. If either side has it off, SHM isn't used on that transport |
| `mode` | `lazy` | `lazy`: set up SHM on first use (faster startup, extra latency on the first SHM message). `init`: set up when the session opens |
| `transport_optimization/enabled` | `true` | Copy large regular payloads into SHM automatically on SHM-capable transports |
| `transport_optimization/pool_size` | 16 MiB | Size of the pool used for those copies (must be non-zero) |
| `transport_optimization/message_size_threshold` | 3072 | Only payloads at least this big are copied |
| `transport_optimization/messages` | `put`, `query`, `reply` | Which message types get the implicit copy. `delete` has no payload, so listing it does nothing |

Without the `shared-memory` feature this whole section is accepted and **ignored**.

## Tuning

- **Latency-sensitive startup**: `mode: "init"` so the first large message doesn't pay for setup.
- **Many large concurrent messages**: raise `pool_size`, otherwise payloads that don't fit fall back to
  normal transmission.
- **Explicit SHM only**: `transport_optimization: { enabled: false }`. Only buffers you allocate yourself
  go through SHM.
- **RPC-style traffic under high CPU load**: `messages: ["put"]` keeps queries and replies off the implicit
  SHM path. This is the workaround for issue [#2628](https://github.com/eclipse-zenoh/zenoh/issues/2628)
  (the watchdog invalidating in-transit buffers).

## Verifying SHM is used

- Admin space: `@/<zid>/<mode>` → `sessions[].shm: true` for the transport.
- API (:material-flask: unstable): `Transport::is_shm()`.
- Stats: the `shm="true"` label on `*_network_message*` metrics ([Statistics](../configuration/stats.md)).

## Sources

- `commons/zenoh-config/src/lib.rs` (`ShmConf`, `LargeMessageTransportOpt`), `defaults.rs`
- `DEFAULT_CONFIG.json5`
