# Downsampling

Downsampling caps the **rate** of messages crossing a transport, per key-expression rule. Typical uses: keep
a 1 kHz sensor stream from flooding a Wi-Fi uplink, or limit how often telemetry leaves a robot.

```json5
downsampling: [
  {
    id: "wlan0-egress",                  // optional, unique
    interfaces: ["wlan0"],               // optional, non-empty
    link_protocols: ["tcp", "udp"],      // optional, non-empty
    flows: ["egress"],                   // optional: "ingress", "egress" (default both)
    messages: ["put", "delete"],         // required: "put", "delete", "query", "reply"
    rules: [                             // required, non-empty
      { key_expr: "robot/lidar/**",  freq: 5.0 },   // at most 5 msg/s
      { key_expr: "robot/imu",       freq: 50.0 },
    ],
  },
],
```

## How it works

For each matching transport and direction, each rule keeps **one** timestamp: when it last let a message
through. A message whose key matches the rule is passed only if at least `1 / freq` seconds have gone by
since then. Otherwise it's **dropped**.

!!! warning "The rate is per rule, not per key"
    With `{ key_expr: "robot/lidar/**", freq: 5 }`, the **whole** set of keys under `robot/lidar/` shares
    5 msg/s per transport and direction. If `robot/lidar/front` and `robot/lidar/rear` both publish, they
    compete for those slots. For a separate limit per key, write one rule per key.

Other details:

- If several rules match a key, the first one found in the rule tree is used. Avoid overlapping rules.
- `freq: 0` drops **every** matching message.
- It's a strict "drop if too soon" filter. There's no queue and no smoothing: if 10 messages arrive in a
  burst, the first passes and the next 9 are dropped.
- Only the message types in `messages` are affected. Declarations, interests and final responses always pass.
- A dropped **query** is answered at once with a `ResponseFinal` by the dropping node, so the querier
  doesn't wait for its timeout. It just gets no replies from that route.

## Transport matching

- `interfaces`: **every** link of the transport must be on one of these interfaces.
- `link_protocols`: at least one link must use one of these protocols.
- Rules are created per transport when it opens. Multicast transports aren't affected.

## Observing drops

- The first drop logs `Some message(s) have been dropped by the downsampling interceptor` at INFO, once per
  process.
- Each drop is logged at TRACE with its key and face:
  `RUST_LOG=zenoh::net::routing::interceptor=trace`.
- With the `stats` feature, drops are counted in `*_network_message_dropped_payload` with
  `reason="downsampling"`. See [Statistics](stats.md).

## Validation

- A repeated `id` is an error: `Invalid Downsampling config: id '...' is repeated`.
- `messages` and `rules` must not be empty. Optional lists must not be empty if present.

## Sources

- `commons/zenoh-config/src/lib.rs` (`DownsamplingItemConf`, `DownsamplingRuleConf`)
- `zenoh/src/net/routing/interceptor/downsampling.rs`
