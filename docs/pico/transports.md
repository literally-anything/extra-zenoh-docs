# zenoh-pico transports & links

## Transports

zenoh-pico has three kinds of transport. Which one a session uses depends on the mode and the locator.

| Transport | How you get it | Remote side | Notes |
|---|---|---|---|
| **Unicast, client** | `mode=client` + a connect locator (or scouting) | A Rust router or peer (or a pico peer) | One connection. Connect locators are **alternatives**, tried in order until one works |
| **Unicast, peer** | `mode=peer` + `listen` on TCP/TLS and/or `connect` locators | Other peers (pico or Rust) | Several peers. Up to **10 incoming** connections (`Z_LISTEN_MAX_CONNECTION_NB`); more are refused (`Refusing connection as max connections currently reached`). Needs `Z_FEATURE_UNICAST_PEER` |
| **Multicast, peer** | `mode=peer` + `listen` on a UDP multicast group (or Bluetooth) | Every member of the group | No handshake: members announce themselves with JOIN every 2.5 s |
| **Raw Ethernet, peer** | `mode=peer` + `listen` on `reth/…` | Other raw-Ethernet pico nodes | Ethernet frames with their own EtherType. Linux only. Needs `Z_FEATURE_RAWETH_TRANSPORT` |

In every mode, pico only **sends its own data and receives data for itself**. It doesn't forward anything:
a pico peer connected to two other peers never relays between them. Checked with three pico peers A–B–C in
a line: B received A's publications and C received none.

### Opening rules (`z_open`)

- **Client**: needs at least one connect locator that works. With none configured, it scouts (UDP
  multicast) for a router or peer and connects to the first that answers.
- **Peer**: needs a *primary transport*. That's the listen locator if it opens, otherwise the first connect
  locator that works. The other connect locators are added as extra peers, and a background *add-peers*
  future keeps retrying them for as long as `Z_CONFIG_CONNECT_TIMEOUT_KEY` allows. The default is 0
  (try once), so a peer started before its neighbour **won't** connect later unless you set a timeout.
  Checked: a peer started before its target was listening never connected.
- Only **one** listen locator is allowed. More than one makes `z_open` fail.
- Accepting incoming unicast peers needs TCP or TLS. Without either compiled in, a unicast peer can't listen.
- Retry and exit-on-failure keys (unstable) are on the [Configuration](configuration.md) page.

### Session parameters negotiated with Rust nodes

| Item | pico side |
|---|---|
| Protocol version | `0x09` (must match exactly) |
| Batch size | `Z_BATCH_UNICAST_SIZE` = **2048**. The smaller side wins, so a Rust router sends ≤ 2048-byte batches to pico |
| SN / request ID resolution | 32-bit each (`Z_SN_RESOLUTION`, `Z_REQ_RESOLUTION` = `0x02`) |
| Lease | `Z_TRANSPORT_LEASE` = 10 s. Keep-alives every lease / `Z_TRANSPORT_LEASE_EXPIRE_FACTOR` (3) ≈ 3.3 s |
| Extensions offered | **Patch** only (fragment markers), when fragmentation is compiled in. No QoS, SHM, authentication, compression, low latency, multilink or region name |
| ZID length | 16 bytes for generated IDs (`Z_ZID_LENGTH`) |

## Links

| Locator | Feature (default) | Connect | Listen | Transport | Flow | Reliable | MTU | Platforms |
|---|---|---|---|---|---|---|---|---|
| `tcp/<ip>:<port>` | `Z_FEATURE_LINK_TCP` (on) | ✅ | ✅ (peer) | unicast | stream | yes | 65535 | all with TCP ([Platforms](platforms.md)) |
| `udp/<ip>:<port>` | `Z_FEATURE_LINK_UDP_UNICAST` (on) | ✅ | ❌ | unicast | datagram | no | 1450 | all with UDP |
| `udp/<group>:<port>` | `Z_FEATURE_LINK_UDP_MULTICAST` (on) | ❌ | ✅ | multicast | datagram | no | 1450 | all with UDP except FreeRTOS-Plus-TCP |
| `tls/<host>:<port>` | `Z_FEATURE_LINK_TLS` (off) | ✅ | ✅ (peer) | unicast | stream | yes | 65535 | Unix only, needs mbedtls |
| `ws/<host>:<port>` | `Z_FEATURE_LINK_WS` (off) | ✅ | ❌ | unicast | datagram | yes | 65535 | Emscripten only |
| `serial/<dev>` or `serial/<tx>.<rx>` | `Z_FEATURE_LINK_SERIAL` (off) | ✅ | ❌ | unicast | datagram | no | 1500 | see [Serial](#serial) |
| `bt/<name>` | `Z_FEATURE_LINK_BLUETOOTH` (off) | ❌ | ✅ | **multicast** | stream | no | 128 | Arduino-ESP32 |
| `reth/<mac>` | `Z_FEATURE_RAWETH_TRANSPORT` (off) | ❌ | ✅ | raw Ethernet | datagram | no | — | Linux |

The MTU is the link's limit. The batch actually used is the smaller of that and `Z_BATCH_*_SIZE` (2048).
Endpoint options go after `#`, separated by `;`, as in Rust ([Endpoints](../configuration/endpoints.md)).

### TCP

```text
tcp/192.168.1.10:7447#tout=2000
```

- `tout`: socket timeout in ms (default `Z_CONFIG_SOCKET_TIMEOUT`, 100).
- `TCP_NODELAY` is set unless `Z_FEATURE_TCP_NODELAY=0`.
- A listening peer accepts up to 10 connections (also the `listen()` backlog).

### UDP unicast and multicast

```text
udp/192.168.1.10:7447                     # unicast (client side only)
udp/224.0.0.225:7447#iface=eth0           # multicast group (peer, listen)
udp/224.0.0.225:7447#iface=eth0;join=224.0.0.226|224.0.0.227
```

- `iface`: interface name (or address, depending on platform) for multicast and scouting.
- `join`: extra multicast groups to receive from, separated by `|`.
- `tout`: socket timeout in ms.
- The MTU is fixed at **1450** (`@TODO` in the source: it doesn't depend on the platform yet).
- pico sends multicast batches of at most min(link MTU, `Z_BATCH_MULTICAST_SIZE`), which is **1450**
  bytes with the defaults. Rust peers in the same group must keep `transport/multicast/qos/enabled: false`
  (their default) and use a compatible batch size. See
  [Limitations](limitations.md#multicast-groups-with-rust-peers).

Checked: a pico client reached `zenohd` over `udp/127.0.0.1:…` (all 3 publications arrived). Multicast
couldn't be tested in the sandbox used for these docs (no multicast route).

### Serial

```text
serial//dev/ttyUSB0#baudrate=115200      # POSIX device path (note the double slash)
serial/UART_1#baudrate=921600             # ESP-IDF / Arduino-ESP32 port name
serial/17.16#baudrate=115200              # TX pin . RX pin
serial/usb#baudrate=115200                # Raspberry Pi Pico USB CDC (Z_FEATURE_LINK_SERIAL_USB)
```

`baudrate` is **required**. The other options in the header (data bits, parity, stop bits, flow control,
`tout`) are commented out, so they aren't supported.

| Platform | Device names | Pins |
|---|---|---|
| Linux/macOS/BSD (`tty_posix.c`) | any tty path | ❌ |
| Zephyr | device-tree name (`device_get_binding`) | ❌ |
| ESP-IDF, Arduino-ESP32 | `UART_0`, `UART_1`, `UART_2` | ✅ |
| Raspberry Pi Pico | `uart0_0` (0.1), `uart1_0` (4.5), `uart1_1` (8.9), `uart0_1` (12.13), `uart0_2` (16.17), `usb` | ✅ (those pairs only) |
| Mbed | ❌ | ✅ |
| Flipper Zero | `usart`, `lpuart` | ❌ |
| ThreadX STM32 | name ignored: uses the UART set by `ZENOH_HUART` | ❌ |

The framing matches the Rust `serial` link (`z-serial`): COBS-encoded frames with a 1-byte header, a 2-byte
length, at most 1500 bytes of data and a CRC32, plus an INIT/ACK/RESET handshake retried every 250 ms. pico
only **connects** over serial, so the host side listens:
`zenohd -l "serial//dev/ttyACM0#baudrate=115200"` (built with `--features zenoh/transport_serial`).

Checked: a pico client connected through a pty pair to `zenohd` listening on `serial//dev/pts/0`, and a Rust
subscriber received all of its publications.

`Z_FEATURE_LINK_SERIAL_USB` (Pico USB CDC) needs `Z_FEATURE_UNSTABLE_API`, otherwise CMake turns it off
with a warning.

### TLS

```text
tls/router.example.com:7447
```

TLS is built on mbedtls 2.x or 3.x (4.x is rejected at configure time). Certificates and keys are set with
the `Z_CONFIG_TLS_*` config keys ([Configuration](configuration.md#tls)) or per endpoint after `#`. The
endpoint keys are `root_ca_certificate`, `listen_private_key`, `listen_certificate`, `enable_mtls`,
`connect_private_key`, `connect_certificate` and `verify_name_on_connect`, plus `*_base64` variants. These
are the names Rust uses in its global `transport/link/tls` section, **not** the `*_file` names Rust uses on
endpoints.

- Server-authenticated TLS to a Rust `tls/` listener: **works** (checked with mbedtls 2.28.8, hostname
  verification on).
- **Mutual TLS to a Rust listener fails with mbedtls 2.x.** Rust requires **TLS 1.3** when mTLS is on, and
  mbedtls 2.28 only offers TLS 1.2. The router logs `peer is incompatible: SupportedVersionsExtensionRequired`.
  mbedtls 3.x with TLS 1.3 enabled should negotiate it, but that wasn't tested.

### WebSocket (browser)

`ws/` exists only for the Emscripten (WebAssembly) platform, which uses the browser's WebSocket API. It's
meant to connect to a Rust node's `ws/` listener (not tested here). It's not the zenoh-ts remote-api
protocol.

### Bluetooth

```text
bt/my-device#mode=master;profile=spp;tout=1000
```

Bluetooth Classic **SPP** on Arduino-ESP32 (`BluetoothSerial`). `mode` is `master` or `slave`, `profile`
must be `spp`, and `tout` is a timeout in ms. It's a **multicast-type** transport with a 128-byte MTU,
meant for pico-to-pico links. Rust Zenoh has no Bluetooth link.

### Raw Ethernet

```text
reth/30:03:8c:c8:00:a1#iface=eth0;ethtype=0x72e0;whitelist=30:03:8c:c8:00:a2,;mapping=sensors/**#aa:bb:cc:dd:ee:ff#0010,
```

| Option | Default | Meaning |
|---|---|---|
| address | — | This node's **source MAC** |
| `iface` | `lo` | Interface |
| `ethtype` | `0x72e0` | EtherType (must be ≥ `0x600`) |
| `mapping` | one default entry with destination `aa:bb:cc:dd:ee:ff`, no VLAN | Comma-separated `keyexpr#dest-mac#vlan` entries (VLAN optional, hex). Each message goes to the MAC (and VLAN) of the first entry whose key expression intersects its key, or to the first entry if none does |
| `whitelist` | none | Comma-separated source MACs accepted on receive |

It needs `CAP_NET_RAW` (an `AF_PACKET` socket) and works only between pico nodes. Two pico peers on `lo`
opened their sessions in testing, but no data arrived, so treat this transport as experimental.

## Scouting

With `Z_FEATURE_SCOUTING` (needs UDP unicast), a client with no connect locator sends SCOUT to
`Z_CONFIG_MULTICAST_LOCATOR_KEY` (`udp/224.0.0.224:7446`). It looks for the kinds in
`Z_CONFIG_SCOUTING_WHAT_KEY` (default `3` = router | peer) for `Z_CONFIG_SCOUTING_TIMEOUT_KEY` ms (**1000**
by default, against 3000 in Rust). It connects to the first HELLO it gets. `z_scout()` gives you the HELLOs
directly. Scouting uses at most `Z_MAX_NUM_SCOUT_INTERFACES` (10) interfaces. pico doesn't answer scouts and
doesn't gossip.

## Sources

- `zenoh-pico@1.10.1`: `src/link/link.c` (open/listen dispatch), `src/link/unicast/*.c`,
  `src/link/multicast/*.c` (capabilities, MTU, options), `include/zenoh-pico/link/config/*.h`
- `src/link/transport/upper/serial_protocol.c`, `include/zenoh-pico/link/transport/serial_protocol.h`,
  `src/link/transport/serial/*`
- `src/link/transport/upper/tls_stream.c`, `CMakeLists.txt` (mbedtls), `src/transport/raweth/link.c`
- `src/transport/manager.c`, `src/transport/unicast/accept.c`, `src/net/session.c`, `include/zenoh-pico/config.h.in`
- `src/protocol/codec/transport.c` (INIT extensions)
- Live tests: zenoh-pico 1.10.1 (Linux build) against `zenohd` 1.10.1 over TCP, UDP, serial (pty), TLS and mTLS
