# zenoh-pico limitations & interoperability

What zenoh-pico 1.10.1 doesn't do, what behaves differently from Rust Zenoh, and what to set on the Rust side
when the two meet. Items marked **checked** were reproduced on Linux against `zenohd` 1.10.1. The rest
come from the source.

## Network role

### No routing

A pico node only sends its own data and receives data for itself. It never forwards between its transports,
even in peer mode with several peers. **Checked**: with pico peers A → B → C connected in a line, B received
A's publications and C received none. Put a Rust router (or Rust peer) in the middle for any multi-hop path.

- There's **no router mode** (`mode` is `client` or `peer`).
- Pico peers in **unicast** form a mesh only by explicit configuration: one listen locator per node plus
  connect locators to the others. Each node accepts at most **10** incoming peers.
- pico **doesn't answer scouting** and doesn't gossip. It sends SCOUT (as a client looking for a gateway) but
  never replies with HELLO. Rust peers can't discover pico peers, so configure `connect/endpoints`
  explicitly towards the pico listener.

### Starting order matters in peer mode

A peer's connect locators are tried **once** by default (`Z_CONFIG_CONNECT_TIMEOUT_KEY` = 0). If the target
isn't listening yet, that peer is never connected. **Checked.** Set the (unstable) connect timeout to a
positive value or `-1` to keep retrying.

## Message sizes

| Limit | Default | Consequence |
|---|---|---|
| Receive buffer for reassembled messages (`FRAG_MAX_SIZE`) | **4096 bytes** | Bigger messages sent **to** pico are silently dropped. **Checked**: 4000 bytes delivered, 5000 bytes not |
| Batch (`BATCH_UNICAST_SIZE`) | 2048 bytes | Negotiated with Rust, so Rust nodes send pico batches of ≤ 2048 bytes. Larger messages are fragmented |
| UDP MTU | 1450 bytes | Fixed in the source for every platform |
| Serial MTU | 1500 bytes | Same as Rust's serial link |
| Bluetooth MTU | 128 bytes | |

Protect devices on the Rust side: add a [low-pass filter](../configuration/low-pass-filter.md) on the router
interface that faces them (`size_limit` ≤ `FRAG_MAX_SIZE`), so oversized data doesn't use up the link only to
be dropped.

## Features missing compared with Rust

| Missing | Effect when talking to Rust nodes |
|---|---|
| **User/password and public-key authentication** | `Z_CONFIG_USER_KEY`/`PASSWORD_KEY` exist but are never read. A pico node **can't join a router that requires `usrpwd`**. **Checked**: `z_open` failed with `Unable to open session!`. Use TLS/mTLS for authentication instead |
| ACL, downsampling, QoS overwrite, low-pass (interceptors) | Not available on the device. Configure them on the router facing it |
| QoS / priority queues | pico never negotiates the QoS extension, so Rust uses a **single queue** on transports to pico. Priorities are still carried per message, but nothing is reordered |
| Shared memory, compression, low latency, multilink | Not negotiated. The transport uses plain batches |
| Regions | pico doesn't send a region name. Rust gateway filters on `region_names` won't match pico nodes; use `modes` or `zids` |
| Automatic timestamps | `Z_CONFIG_ADD_TIMESTAMP_KEY` is unused. pico data has no timestamp unless you set one (`z_timestamp_new`) or a Rust router adds it (routers timestamp by default) |
| Storages, plugins, REST, admin-space writes | None |
| Full admin space | Optional and small: `@/<zid>/pico/**` (see [architecture](architecture.md#admin-space)) |
| Gossip, scouting replies | See above |

## Security

- **TLS** works on Unix builds with mbedtls. **Checked** against a Rust `tls/` listener with hostname
  verification.
- **mTLS to a Rust listener fails with mbedtls 2.x**: Rust only accepts TLS 1.3 when mTLS is on, and
  mbedtls 2.28 offers TLS 1.2 (`peer is incompatible: SupportedVersionsExtensionRequired` on the router).
  **Checked.** Use mbedtls 3.x with TLS 1.3 enabled (untested), or terminate mTLS elsewhere.
- No TLS on RTOS platforms at 1.10.1 (the TLS socket type exists only for Unix).

## Threads and timing

- Multi-thread builds use **one** executor thread. Callbacks run on it, so a slow callback delays keep-alives
  and lease checks. Past the remote's lease (10 s by default) the remote closes the session.
- Single-thread builds must call `zp_spin_once` often: at least several times per keep-alive interval
  (lease / 3 ≈ 3.3 s).
- OpenCR has no thread support: build it with `Z_FEATURE_MULTI_THREAD=0`.
- On FreeRTOS platforms, `z_realloc` returns `NULL` (not implemented).

## Links

- **Serial**: pico can only *connect*, so the Rust side listens (`zenohd -l serial/…`, built with
  `transport_serial`). **Checked** through a pty pair. `baudrate` is required, and parity, stop bits and flow
  control aren't configurable.
- **UDP unicast**: pico can only connect (client side). **Checked** against a `zenohd` `udp/` listener.
- **WebSocket**: Emscripten only. **Bluetooth**: Arduino-ESP32 only, and only between pico nodes.
- **Raw Ethernet**: Linux only, needs `CAP_NET_RAW`, pico-to-pico only. Two pico peers on `lo` opened their
  sessions but exchanged no data in testing. Treat it as experimental.
- **FreeRTOS-Plus-TCP**: no UDP multicast (forced off at configure time).

## Multicast groups with Rust peers

A UDP multicast group has no handshake, so every member must agree on the parameters:

- Keep Rust's `transport/multicast/qos/enabled: false` and `compression/enabled: false` (both defaults). pico
  never uses QoS or compression.
- pico receives multicast batches into a `BATCH_MULTICAST_SIZE` buffer (2048 bytes) and sends batches of at
  most 1450 bytes (the UDP MTU in pico). Set `transport/link/tx/batch_size: 2048` (or smaller) on the Rust
  members of the group. Otherwise a Rust node may send batches pico can't read.
- Multicast couldn't be tested in the sandbox used for these docs, so this section comes from the source only.

## Build system gotchas

- **`MinSizeRel` and `RelWithDebInfo` produce `-O0` builds**: only `Release` gets the optimisation flags.
  Other build types fall into the debug branch, which appends `-g -O0 -Werror`. Seen in
  `compile_commands.json`.
- In those non-release builds, `-Werror` makes some aggressive feature combinations fail to compile
  (unused variables). See [Capabilities](capabilities.md#feature-flags).
- Feature flags are cached by CMake. Wipe the build directory when you change them.
- `ZP_SYSTEM_LAYER` was replaced by `ZP_PLATFORM` (setting the old one is a configure error).
- The Zephyr module's Kconfig options have no defaults, so every feature is off until you enable it in
  `prj.conf`.

## Interoperability checklist (Rust side)

```json5
// zenohd facing pico devices
{
  mode: "router",
  listen: { endpoints: ["tcp/0.0.0.0:7447", "serial//dev/ttyACM0#baudrate=115200"] },   // serial needs transport_serial
  // no transport/auth/usrpwd on links pico uses: pico can't authenticate that way
  low_pass_filter: [
    { id: "protect-pico", interfaces: ["eth1"], flows: ["egress"],
      messages: ["put", "delete", "reply"], key_exprs: ["**"], size_limit: 4096 },
  ],
  transport: {
    multicast: { qos: { enabled: false }, compression: { enabled: false } },   // defaults, if you use multicast
  },
}
```

## Sources

- `zenoh-pico@1.10.1`: `src/session/rx.c` (no forwarding), `src/session/scout.c` (no HELLO replies),
  `src/net/config.c` (keys read), `src/protocol/codec/transport.c` (INIT extensions),
  `src/transport/unicast/accept.c` (10-peer limit), `src/link/*`, `CMakeLists.txt`, `zephyr/Kconfig.zenoh`
- `eclipse-zenoh/zenoh@173b1220`: `io/zenoh-links/zenoh-link-tls/src/utils.rs` (TLS 1.3-only with mTLS)
- Live tests: zenoh-pico 1.10.1 (Linux, multi-thread, single-thread, TLS builds) against `zenohd` 1.10.1
