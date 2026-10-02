# Low-pass filter

The low-pass filter drops messages whose **size** is over a limit, per key expression. Use it to keep large
payloads (images, point clouds) off constrained links, or to protect a router from oversized messages.

```json5
low_pass_filter: [
  {
    id: "no-big-stuff-on-wifi",          // optional, unique
    interfaces: ["wlan0"],               // optional, non-empty
    link_protocols: ["tcp", "udp"],      // optional, non-empty
    flows: ["egress", "ingress"],        // optional (default both)
    messages: ["put", "delete", "query", "reply"],  // required, non-empty
    key_exprs: ["camera/**"],            // required, non-empty
    size_limit: 8192,                    // bytes, inclusive
  },
],
```

## What gets measured

The size checked is **payload + attachment** (serialized), compared against the limit **inclusively**
(`size ≤ size_limit` passes):

| Message | Size counted |
|---|---|
| put | payload + attachment |
| delete | attachment only (no payload) |
| query | query body payload (if any) + attachment |
| reply (put) | payload + attachment |
| reply (delete) | attachment |
| reply error | error payload |

Declarations, interests, final responses and OAM messages always pass.

## Combining limits

All `low_pass_filter` items together build **one** interceptor:

- For a message, every entry whose `key_exprs` **include** the message's key and whose subject (interface ×
  link protocol) matches the transport is considered, and the **smallest** `size_limit` applies.
- If no entry matches, the message passes.

## Transport matching

Unlike downsampling and QoS overwrite, a low-pass entry applies if **any** link of the transport is on a
listed interface (and uses a listed protocol). Interfaces and protocols are combined as a cartesian product,
and leaving either out means "any".

## What happens to dropped messages

- A dropped **put/delete** is gone, with no error to the publisher.
- A dropped **query** never reaches the queryable. The querier gets no replies from that route and waits
  for its timeout.
- A dropped **reply** is missing from the querier's results.

With the `stats` feature, drops are counted with `reason="low-pass"`.

## Sources

- `commons/zenoh-config/src/lib.rs` (`LowPassFilterConf`)
- `zenoh/src/net/routing/interceptor/low_pass.rs`
