# zenoh-pico

zenoh-pico is a **separate C implementation** of the Zenoh protocol for microcontrollers and other
constrained systems. Its API is the same `z_` API as zenoh-c, so code moves between them easily, but the
implementation, limits and build options are its own.

This page is the short version. The [zenoh-pico section](../../pico/index.md) covers it in full:
[architecture](../../pico/architecture.md), [platforms (RTOS)](../../pico/platforms.md),
[transports](../../pico/transports.md), [capabilities and feature flags](../../pico/capabilities.md),
[configuration](../../pico/configuration.md) and [limitations](../../pico/limitations.md).

## At a glance

- **Modes**: client (one connection to a router or peer) or peer (unicast peers and/or a UDP multicast
  group). No router mode, and **no forwarding** between peers.
- **Platforms**: Linux, macOS, BSD, Windows, Zephyr, FreeRTOS (lwIP or Plus-TCP), ESP-IDF, Arduino (ESP32,
  OpenCR), Mbed, Raspberry Pi Pico, ThreadX (STM32), Flipper Zero, Emscripten.
- **Links**: TCP, UDP unicast and multicast, serial, TLS (Unix), WebSocket (Emscripten), Bluetooth
  (Arduino-ESP32), raw Ethernet (Linux), each behind a `Z_FEATURE_LINK_*` flag.
- **Size limits** (compile time): batches of 2048 bytes, and at most **4096 bytes** per received message by
  default.
- **Not available**: shared memory, user/password and public-key authentication, priority queues,
  compression.

## Configuration

There's no JSON5. Use `zp_config_insert(z_loan_mut(config), Z_CONFIG_X_KEY, "value")` with `MODE`, `CONNECT`,
`LISTEN`, `MULTICAST_SCOUTING`, `MULTICAST_LOCATOR`, `SCOUTING_TIMEOUT`, `SCOUTING_WHAT`, `SESSION_ZID`, the
`TLS_*` keys and (unstable) connect/listen timeout and exit-on-failure keys. `Z_CONFIG_USER_KEY`,
`Z_CONFIG_PASSWORD_KEY` and `Z_CONFIG_ADD_TIMESTAMP_KEY` are defined but **unused**. Full table:
[pico configuration](../../pico/configuration.md).

## Threads

- With `Z_FEATURE_MULTI_THREAD=1` (default), one background executor thread runs reading, leases,
  keep-alives and accepting, and calls your callbacks. It starts with `z_open`. `zp_start_read_task` and
  `zp_start_lease_task` are deprecated.
- With `Z_FEATURE_MULTI_THREAD=0`, call **`zp_spin_once(z_loan(session))`** in your main loop. `zp_read`,
  `zp_send_keep_alive` and `zp_send_join` are deprecated in its favour.

## Pico-specific APIs (`zp_*`)

`zp_config_insert`/`zp_config_get`, `zp_spin_once`, `zp_batch_start`/`zp_batch_flush`/`zp_batch_stop`
(explicit batching), `zp_hello_locators`, and `zp_start_admin_space`/`zp_stop_admin_space` (with
`Z_FEATURE_ADMIN_SPACE`, unstable).

## Sources

- `zenoh-pico@1.10.1`: `include/zenoh-pico/api/primitives.h`, `include/zenoh-pico/config.h.in`, `CMakeLists.txt`,
  `cmake/platforms/`, `docs/config.rst`
