# zenoh-pico

zenoh-pico is a **separate C implementation** of the Zenoh protocol for microcontrollers and other
constrained systems. Its API is the same `z_` API as zenoh-c, so code moves between them easily, but the
implementation, limits and build options are its own.

## Platforms

Ports in `src/system/`: Unix (Linux/macOS), Windows, Zephyr, FreeRTOS, ESP-IDF, Arduino, Mbed, Raspberry Pi
Pico (`rpi_pico`), ThreadX, Flipper Zero, Emscripten (WebAssembly).

## Modes and links

- **client** (connects to a router or peer) or **peer** (unicast peer or UDP multicast group).
- Links: TCP, UDP unicast, UDP multicast, serial, serial-USB, TLS, WebSocket, Bluetooth, raw Ethernet,
  each behind a build flag.

## Build features

Set at CMake time (`-DZ_FEATURE_X=0/1`). Defaults from `CMakeLists.txt` at 1.10.1:

| Flag | Default | | Flag | Default |
|---|---|---|---|---|
| `Z_FEATURE_PUBLICATION` | 1 | | `Z_FEATURE_LINK_TCP` | 1 |
| `Z_FEATURE_SUBSCRIPTION` | 1 | | `Z_FEATURE_LINK_UDP_UNICAST` | 1 |
| `Z_FEATURE_QUERY` | 1 | | `Z_FEATURE_LINK_UDP_MULTICAST` | 1 |
| `Z_FEATURE_QUERYABLE` | 1 | | `Z_FEATURE_LINK_SERIAL` | 0 |
| `Z_FEATURE_LIVELINESS` | 1 | | `Z_FEATURE_LINK_SERIAL_USB` | 0 |
| `Z_FEATURE_MATCHING` | 1 | | `Z_FEATURE_LINK_TLS` | 0 |
| `Z_FEATURE_INTEREST` | 1 | | `Z_FEATURE_LINK_WS` | 0 |
| `Z_FEATURE_SCOUTING` | 1 | | `Z_FEATURE_LINK_BLUETOOTH` | 0 |
| `Z_FEATURE_FRAGMENTATION` | 1 | | `Z_FEATURE_RAWETH_TRANSPORT` | 0 |
| `Z_FEATURE_BATCHING` | 1 | | `Z_FEATURE_MULTICAST_TRANSPORT` | 1 |
| `Z_FEATURE_MULTI_THREAD` | 1 | | `Z_FEATURE_UNICAST_TRANSPORT` | 1 |
| `Z_FEATURE_UNICAST_PEER` | 1 | | `Z_FEATURE_AUTO_RECONNECT` | 1 |
| `Z_FEATURE_ENCODING_VALUES` | 1 | | `Z_FEATURE_TCP_NODELAY` | 1 |
| `Z_FEATURE_SESSION_CHECK` | 1 | | `Z_FEATURE_LOCAL_SUBSCRIBER` | 0 |
| `Z_FEATURE_LOCAL_QUERYABLE` | 0 | | `Z_FEATURE_MULTICAST_DECLARATIONS` | 0 |
| `Z_FEATURE_ADVANCED_PUBLICATION` | 0 | | `Z_FEATURE_ADVANCED_SUBSCRIPTION` | 0 |
| `Z_FEATURE_CONNECTIVITY` (unstable) | 0 | | `Z_FEATURE_ADMIN_SPACE` | 0 |
| `Z_FEATURE_UNSTABLE_API` | 0 | | `Z_FEATURE_RX_CACHE` | 0 |

Size and timing settings:

| Setting | Default | Meaning |
|---|---|---|
| `BATCH_UNICAST_SIZE` | **2048** | Max unicast batch |
| `BATCH_MULTICAST_SIZE` | **2048** | Max multicast batch. **All members of a multicast group must use the same `batch_size`**, so set `transport/link/tx/batch_size: 2048` on Rust nodes in a pico group |
| `FRAG_MAX_SIZE` | **4096** | Largest reassembled message. Larger messages to a pico node are dropped |
| `Z_TRANSPORT_LEASE` | 10000 ms | Lease announced |
| `Z_TRANSPORT_LEASE_EXPIRE_FACTOR` | 3 | |
| `Z_CONFIG_SOCKET_TIMEOUT` | 100 ms | |

!!! tip "Talking to pico from a Rust router"
    Pico's 4 KiB reassembly limit means large payloads never reach the device. Protect it with a
    [low-pass filter](../../configuration/low-pass-filter.md) on the router (egress towards the device's
    interface) so oversized messages are dropped before they use the link.

## Configuration

There's no JSON5. Use `zp_config_insert(z_loan_mut(config), Z_CONFIG_X_KEY, "value")` with:

`Z_CONFIG_MODE_KEY`, `Z_CONFIG_CONNECT_KEY`, `Z_CONFIG_LISTEN_KEY`, `Z_CONFIG_CONNECT_TIMEOUT_KEY`,
`Z_CONFIG_CONNECT_EXIT_ON_FAILURE_KEY`, `Z_CONFIG_LISTEN_TIMEOUT_KEY`, `Z_CONFIG_LISTEN_EXIT_ON_FAILURE_KEY`,
`Z_CONFIG_MULTICAST_SCOUTING_KEY`, `Z_CONFIG_MULTICAST_LOCATOR_KEY`, `Z_CONFIG_SCOUTING_TIMEOUT_KEY`,
`Z_CONFIG_SCOUTING_WHAT_KEY`, `Z_CONFIG_SESSION_ZID_KEY`, `Z_CONFIG_ADD_TIMESTAMP_KEY`, `Z_CONFIG_USER_KEY`,
`Z_CONFIG_PASSWORD_KEY`, and the TLS keys (`Z_CONFIG_TLS_ROOT_CA_CERTIFICATE_KEY`, `…_LISTEN_*`,
`…_CONNECT_*`, `…_ENABLE_MTLS_KEY`, `…_VERIFY_NAME_ON_CONNECT_KEY`).

## Background tasks

With `Z_FEATURE_MULTI_THREAD=1`, the read and lease tasks start automatically when the session opens
(`zp_start_read_task`/`zp_start_lease_task` are deprecated). Single-threaded builds drive the session by
calling `zp_read()` and `zp_send_keep_alive()` (and the related `zp_*` functions) from the main loop.

## Pico-specific APIs

`zp_start_admin_space` / `zp_stop_admin_space` (with `Z_FEATURE_ADMIN_SPACE`, unstable) expose a small admin space for the device.

## Sources

- `zenoh-pico@1.10.1`: `CMakeLists.txt`, `include/zenoh-pico/api/` (`primitives.h`, `liveliness.h`, `admin_space.h`, `advanced_*.h`, `serialization.h`), `include/zenoh-pico/config.h.in`, `src/system/`
