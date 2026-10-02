# zenoh-pico

zenoh-pico is a **separate implementation** of Zenoh, written in C11 for microcontrollers and other
constrained systems. It speaks the same wire protocol (version `0x09`) as the Rust implementation and has
the same `z_*` C API as zenoh-c, but it shares no code with either. Its internals, limits and build
options are its own, and it **implements only part** of what the Rust stack does.

This section documents zenoh-pico **1.10.1** (`eclipse-zenoh/zenoh-pico`, tag `1.10.1`; `main` was at the
1.10.1 release merge when this was written). Claims were checked against the source, and the important
ones were also tested on Linux against a Rust `zenohd` 1.10.1 (see each page).

## What it is, in one table

| | zenoh-pico | Rust Zenoh (`zenohd`, zenoh-c, bindings) |
|---|---|---|
| Language | C11 (C99 subset builds), a little C++ for Arduino/Mbed | Rust |
| Modes | **client**, **peer** | client, peer, **router** |
| Routing | **None**: delivers only to and from its own session | Full (link-state, regions, gateways) |
| Threads | 1 background executor thread, or **none** (`zp_spin_once`) | Several Tokio pools |
| Links | TCP, UDP (unicast and multicast), serial, TLS (Unix), WebSocket (browser), Bluetooth (ESP32), raw Ethernet (Linux) | 10 link types |
| Batch size | **2048 bytes** by default (compile time) | 65535 |
| Largest message received | **4096 bytes** by default (compile time) | 1 GiB |
| QoS on the wire | Priorities carried per message, but **no per-priority queues or channels** | 8 priority queues |
| Security | TLS and mTLS (Unix, mbedtls). **No user/password, no public-key auth, no ACL** | All of them |
| Shared memory, compression, multilink, low latency | ❌ | ✅ |
| Admin space | Optional, small (`@/<zid>/pico/**`) | Full |
| Config | Small key/value table (`zp_config_insert`) | Full JSON5 tree |

## Pages

| Page | Covers |
|---|---|
| [Architecture](architecture.md) | Layers, source tree, the executor, the send and receive paths, memory and ownership |
| [Supported platforms (RTOS)](platforms.md) | Every platform profile, its RTOS, threading and links, building, porting to a new one |
| [Transports & links](transports.md) | Client/peer/multicast/raw-Ethernet transports, every link with locator syntax, MTU and options |
| [Capabilities & feature flags](capabilities.md) | What the API can do, every `Z_FEATURE_*` flag with defaults and dependencies, footprint |
| [Configuration](configuration.md) | Runtime keys (`zp_config_insert`), compile-time settings, CMake options |
| [Limitations & interoperability](limitations.md) | What's missing or different, and what to configure on the Rust side |

The [zenoh-pico API page](../api/languages/pico.md) in the API section gives the short version.

## Quick start (Linux)

```bash
git clone https://github.com/eclipse-zenoh/zenoh-pico && cd zenoh-pico
git checkout 1.10.1
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
./build/examples/z_sub -m client -e tcp/127.0.0.1:7447     # needs a zenohd on 7447
./build/examples/z_pub -m client -e tcp/127.0.0.1:7447
```

The examples take `-m client|peer`, `-e <connect locator>`, `-l <listen locator>`, `-k <key>` and
(publishers) `-n <count>`.

## Sources

- `zenoh-pico@1.10.1`: `README.md`, `CMakeLists.txt`, `include/zenoh-pico/config.h.in`, `docs/*.rst`,
  `cmake/platforms/*.cmake`, `src/`, `examples/unix/c11/`
