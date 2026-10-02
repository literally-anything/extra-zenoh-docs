# QoS overwrite

Zenoh can change message QoS (priority, congestion control, express, reliability) from configuration,
without touching application code. There are two mechanisms, and they work at different levels.

| | `qos/publication` | `qos/network` |
|---|---|---|
| Where | In the **publishing session** | On **transports** (ingress or egress) of any node, routers included |
| What | Publisher defaults: put/delete from `session.put`/`delete` and declared publishers | Each put, delete or query as it crosses a link |
| Matches on | Key expression only | Key, message type, peer ZID, interface, link protocol, current QoS, payload size |
| Cost | None at send time (resolved when the publisher is declared) | Checked per message (with a per-key cache) |
| Can set | `congestion_control`, `priority`, `express`, :material-flask: `reliability`, :material-flask: `allowed_destination` | `congestion_control`, `priority` (absolute or relative), `express` |

## `qos/publication`

```json5
qos: {
  publication: [
    {
      key_exprs: ["robot/telemetry/**"],
      config: {
        congestion_control: "drop",
        priority: "data_low",
        express: false,
        // reliability: "best_effort",        // unstable
        // allowed_destination: "remote",     // unstable: "session_local" | "remote" | "any"
      },
    },
    {
      key_exprs: ["robot/estop"],
      config: { priority: "real_time", congestion_control: "block", express: true },
    },
  ],
},
```

How it's applied:

- When a publisher is declared, or when `session.put()`/`delete()` runs, Zenoh looks for config entries
  whose `key_exprs` **include** the publisher's key expression.
- Fields set in the matching entry **override** what the application asked for in the builder. Fields not
  set keep the application's value.
- If several entries include the key, the first match is used and a warning is logged:
  `Publisher declared on ... is included by multiple key_exprs in qos config`.
- `congestion_control` also accepts :material-flask: `block_first`.

Priority names: `real_time`, `interactive_high`, `interactive_low`, `data_high`, `data`, `data_low`, `background`.

## `qos/network`

```json5
qos: {
  network: [
    {
      id: "wifi-bulk-to-background",       // optional, must be unique
      zids: ["38a4829bce9166ee"],          // optional: only transports to these peers
      interfaces: ["wlan0"],               // optional: only transports whose links are on these interfaces
      link_protocols: ["tcp", "quic"],     // optional: only transports with at least one link of these protocols
      messages: ["put", "delete", "query"],// required, non-empty
      flows: ["egress"],                   // optional: "ingress", "egress" (default both)
      key_exprs: ["camera/**"],            // optional: default all keys
      qos: {                               // optional filter on the message's current QoS
        congestion_control: "drop",
        priority: "data",
        express: false,
        reliability: "reliable",
      },
      payload_size: "100000..",            // optional inclusive range "min..max", either side may be empty
      overwrite: {
        priority: "background",            // or a relative change, e.g. +1
        congestion_control: "drop",
        express: false,
      },
    },
  ],
},
```

### Matching

The rule decides which **transports** it applies to when the transport is created:

- `zids`: the remote's ZID must be in the list.
- `interfaces`: **every** link of the transport must be on one of the listed interfaces.
- `link_protocols`: **at least one** link must use one of the listed protocols (`tcp`, `udp`, `tls`,
  `quic`, `ws`, `serial`, `unixpipe`, `unixsock-stream`, `vsock`).

Then, for each message on that transport:

1. Its type must be in `messages`. **Replies can't be overwritten** (only `put`, `delete` and `query` are accepted).
2. If `key_exprs` is set, one of them must **include** the message's key.
3. If `qos` is set, every field given must equal the message's current value.
4. If `payload_size` is set, the payload length must be within the range. Puts and replies use the payload;
   queries use the query body; deletes count as 0.

Each item is its own interceptor, applied in list order, so later items see the QoS that earlier items wrote.

### Overwrite values

- `priority`: a name (`"data_high"`) sets that priority. An **integer** from −7 to +7 changes it **relative**
  to the current one, as a number. Since 1 is the highest data priority and 7 the lowest, `+1` moves
  `data` (5) to `data_low` (6). Results past `background` (7) are clamped to `background`.
- `congestion_control`: `drop` or `block` (:material-flask: `block_first`).
- `express`: `true`/`false`.

!!! bug "Negative increments do nothing in 1.10.1"
    In `qos_overwrite.rs`, the branch for negative increments sits behind `if 1 >= 0 { ... } else { ... }`.
    That condition is always true, so a negative value goes down the positive path, fails the `u8`
    conversion, and adds 0. **`priority: -1` (raising the priority) has no effect.** Write an absolute
    priority name instead.

### Where it doesn't apply

- Multicast transports (no interceptor is created).
- Messages between entities in the same session.

### Validation

- A repeated `id` is an error: `Invalid Qos Overwrite config: id '...' is repeated`.
- An empty list in any optional filter (for example `interfaces: []`) is a parse error. Leave the key out instead.

## Runtime changes

`qos/network` items have `id`s so they can be addressed with `qos/network/id=<id>`. On a running session,
however, only `plugins/**` keys are writable, so these rules are effectively set at startup. See
[Dynamic changes](dynamic-changes.md).

## Sources

- `commons/zenoh-config/src/qos.rs`, `lib.rs` (`QosOverwriteItemConf`)
- `zenoh/src/api/session.rs` (`get_publisher_qos_overwrite`), `api/builders/publisher.rs` (`apply_qos_overwrites`)
- `zenoh/src/net/routing/interceptor/qos_overwrite.rs`
