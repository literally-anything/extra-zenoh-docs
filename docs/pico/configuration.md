# zenoh-pico configuration

zenoh-pico has no JSON5 configuration. Settings come from three places:

1. **Runtime keys**, a small table filled with `zp_config_insert()` before `z_open()`;
2. **Generated compile-time settings**, CMake variables written into `include/zenoh-pico/config.h`;
3. **Manual compile-time settings**, constants you edit in `include/zenoh-pico/config.h.in`.

## Runtime keys

```c
z_owned_config_t config;
z_config_default(&config);
zp_config_insert(z_loan_mut(config), Z_CONFIG_MODE_KEY, "client");
zp_config_insert(z_loan_mut(config), Z_CONFIG_CONNECT_KEY, "tcp/192.168.1.10:7447");
z_owned_session_t s;
z_open(&s, z_move(config), NULL);
```

Values are strings. `zp_config_get()` reads them back.

| Key (ID) | Values | Default | Notes |
|---|---|---|---|
| `Z_CONFIG_MODE_KEY` (0x40) | `"client"`, `"peer"` | `client` | No router mode |
| `Z_CONFIG_CONNECT_KEY` (0x41) | locator | none | Insert several times for several locators: alternatives in client mode, extra peers in peer mode |
| `Z_CONFIG_LISTEN_KEY` (0x42) | locator | none | **One** only. More than one makes `z_open` fail |
| `Z_CONFIG_MULTICAST_SCOUTING_KEY` (0x45) | `"true"`, `"false"` | `true` | |
| `Z_CONFIG_MULTICAST_LOCATOR_KEY` (0x46) | `udp/<ip>:<port>` | `udp/224.0.0.224:7446` | |
| `Z_CONFIG_SCOUTING_TIMEOUT_KEY` (0x47) | ms | **`1000`** | Rust's default is 3000 |
| `Z_CONFIG_SCOUTING_WHAT_KEY` (0x48) | bitmask 1–7 (router 1, peer 2, client 4) | `3` | |
| `Z_CONFIG_SESSION_ZID_KEY` (0x49) | hex ZID | random (16 bytes) | |
| `Z_CONFIG_TLS_*` (0x4B–0x56) | see [TLS](#tls) | | Needs `Z_FEATURE_LINK_TLS` |
| :material-flask: `Z_CONFIG_CONNECT_TIMEOUT_KEY` (0x57) | ms; `0` once, `>0` retry until timeout, `-1` forever | `0` | Needs `Z_FEATURE_UNSTABLE_API` |
| :material-flask: `Z_CONFIG_CONNECT_EXIT_ON_FAILURE_KEY` (0x58) | `"true"`, `"false"` | client `true`, peer `false` | A client always needs one working locator, whatever this says |
| :material-flask: `Z_CONFIG_LISTEN_TIMEOUT_KEY` (0x59) | as connect | `0` | |
| :material-flask: `Z_CONFIG_LISTEN_EXIT_ON_FAILURE_KEY` (0x5A) | `"true"`, `"false"` | `true` | In peer mode with `false`, a failed listen can be replaced by a working connect |
| `Z_CONFIG_USER_KEY` (0x43), `Z_CONFIG_PASSWORD_KEY` (0x44) | — | — | **Defined but unused**: zenoh-pico has no user/password authentication |
| `Z_CONFIG_ADD_TIMESTAMP_KEY` (0x4A) | — | `false` | **Unused** |

### TLS

| Key | Meaning |
|---|---|
| `Z_CONFIG_TLS_ROOT_CA_CERTIFICATE_KEY` / `_BASE64_KEY` | CA bundle (file path / inline base64). Required |
| `Z_CONFIG_TLS_LISTEN_PRIVATE_KEY_KEY` / `_BASE64_KEY` | Listener key (peers that listen on `tls/`) |
| `Z_CONFIG_TLS_LISTEN_CERTIFICATE_KEY` / `_BASE64_KEY` | Listener certificate |
| `Z_CONFIG_TLS_ENABLE_MTLS_KEY` | `true`/`1`/`yes`/`on` to require client certificates |
| `Z_CONFIG_TLS_CONNECT_PRIVATE_KEY_KEY` / `_BASE64_KEY` | Client key (mTLS) |
| `Z_CONFIG_TLS_CONNECT_CERTIFICATE_KEY` / `_BASE64_KEY` | Client certificate (mTLS) |
| `Z_CONFIG_TLS_VERIFY_NAME_ON_CONNECT_KEY` | `false`/`0`/`no`/`off` to skip hostname verification (on by default) |

See [Transports: TLS](transports.md#tls) for the mTLS/TLS 1.3 caveat with Rust routers.

## Generated compile-time settings

Set with `-D<NAME>=<value>` on the CMake command line (or `board_build.cmake_extra_args` in PlatformIO).

| Variable | Default | Meaning |
|---|---|---|
| `BATCH_UNICAST_SIZE` | 2048 | TX/RX buffer per unicast transport, and the batch size offered in INIT |
| `BATCH_MULTICAST_SIZE` | 2048 | Same for multicast (capped by the link MTU: 1450 for UDP) |
| `FRAG_MAX_SIZE` | 4096 | Largest message that can be **received** (defragmentation limit). Bigger ones are dropped |
| `Z_CONFIG_SOCKET_TIMEOUT` | 100 | Default socket timeout (ms) |
| `Z_TRANSPORT_LEASE` | 10000 | Lease announced to remotes (ms) |
| `Z_TRANSPORT_LEASE_EXPIRE_FACTOR` | 3 | Keep-alive interval = lease / factor |
| `Z_TRANSPORT_ACCEPT_TIMEOUT` | 1000 | Peer mode: how long a listener waits for the handshake of an incoming peer (ms) |
| `Z_TRANSPORT_CONNECT_TIMEOUT` | 10000 | How long a connecting node waits for the handshake reply (ms) |
| `Z_RUNTIME_MAX_TASKS` | 64 | Maximum futures in the executor |
| `Z_RUNTIME_IDLE_READ_TASK_SLEEP` | 0 | If > 0, an idle read future sleeps this long (ms) instead of rescheduling at once. That saves CPU, and makes `zp_spin_once` return `false` when idle |
| `Z_FEATURE_*` | — | See [Feature flags](capabilities.md#feature-flags) |
| `ZENOH_LOG` | none | `error`, `warn`, `info`, `debug`, `trace`: log level compiled in |
| `ZENOH_LOG_PRINT` | `printf` | Function used to print logs (for example a UART print) |
| `ZENOH_DEBUG` | — | Legacy level: 1 = error, 2 = info, 3 = debug (overrides `ZENOH_LOG`) |

Other CMake options: `ZP_PLATFORM`, `ZP_EXTERNAL_PACKAGES` ([Platforms](platforms.md)), `BUILD_SHARED_LIBS`
(ON), `BUILD_EXAMPLES` (ON), `BUILD_TOOLS` (OFF), `BUILD_TESTING` (ON), `BUILD_INTEGRATION` (OFF),
`BUILD_FUZZERS` (OFF), `ASAN` (OFF) and `PACKAGING` (OFF, Debian/RPM).

!!! tip "Out of memory at `z_open`?"
    The README's first suggestion for microcontrollers is to lower `BATCH_UNICAST_SIZE`,
    `BATCH_MULTICAST_SIZE` and `FRAG_MAX_SIZE`. Remember that `FRAG_MAX_SIZE` is also the largest message the
    device can receive. Pair it with a [low-pass filter](../configuration/low-pass-filter.md) on the router.

## Manual compile-time settings (`config.h.in`)

| Constant | Value | Meaning |
|---|---|---|
| `Z_ZID_LENGTH` | 16 | Length of generated ZIDs (max 16) |
| `Z_PROTO_VERSION` | `0x09` | Wire protocol version. Don't change it |
| `Z_JOIN_INTERVAL` | 2500 | Multicast JOIN period (ms) |
| `Z_SN_RESOLUTION`, `Z_REQ_RESOLUTION` | `0x02` | 32-bit sequence numbers and request IDs |
| `Z_RX_CACHE_SIZE` | 10 | Entries in the RX cache (`Z_FEATURE_RX_CACHE`) |
| `Z_GET_TIMEOUT_DEFAULT` | 10000 | Default `get` timeout (ms) |
| `Z_LISTEN_MAX_CONNECTION_NB` | 10 | Maximum incoming peers, also used as the `listen()` backlog |
| `Z_MAX_NUM_SCOUT_INTERFACES` | 10 | Interfaces used for scouting |
| `ZP_ASM_NOP` | `__asm__("nop")` | Override if your toolchain lacks `nop` |

`ZENOH_GENERIC` builds skip the generated values and include your own `zenoh_generic_config.h`. That's for
build systems that don't run zenoh-pico's CMake.

## Sources

- `zenoh-pico@1.10.1`: `include/zenoh-pico/config.h.in`, `CMakeLists.txt` (cache variables, logging),
  `include/zenoh-pico/utils/logging.h`, `docs/config.rst`, `README.md` (troubleshooting)
- `src/net/config.c`, `src/net/session.c` (which keys are read)
