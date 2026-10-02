# zenoh-pico capabilities & feature flags

## What the API can do

zenoh-pico implements the zenoh-c API (`z_*`, `ze_*` for extensions, `zp_*` for pico-only functions). At
1.10.1 the only zenoh-c area missing entirely is **shared memory**. Everything else exists, though some
parts are behind feature flags.

| Area | Functions (examples) | Flag(s) |
|---|---|---|
| Session | `z_open`, `z_close`, `z_session_is_closed`, `z_info_zid`, `z_info_routers_zid`, `z_info_peers_zid` | — |
| Config | `z_config_default`, `zp_config_insert`, `zp_config_get` | — |
| Key expressions | `z_keyexpr_from_str`, `z_declare_keyexpr`, `z_keyexpr_canonize`, `_join`, `_concat`, `_intersects`, `_includes`, `_relation_to` | — |
| Payloads | `z_bytes_*` (copy/from-buffer, reader, writer, slice iterator), `ze_serialize_*` / `ze_deserialize_*` | — |
| Encodings | `z_encoding_*` constants and `z_encoding_from_str` | `Z_FEATURE_ENCODING_VALUES` for the constants |
| Publish | `z_put`, `z_delete`, `z_declare_publisher`, `z_publisher_put/delete` (priority, congestion control, express, attachment, encoding, timestamp) | `Z_FEATURE_PUBLICATION` |
| Subscribe | `z_declare_subscriber`, `z_declare_background_subscriber`, closures, `z_fifo_channel_sample_new`, `z_ring_channel_sample_new` | `Z_FEATURE_SUBSCRIPTION` |
| Query | `z_get`, `z_declare_querier`, `z_querier_get`, reply channels | `Z_FEATURE_QUERY` |
| Queryable | `z_declare_queryable` (`complete`), `z_query_reply`, `z_query_reply_err`, `z_query_reply_del` | `Z_FEATURE_QUERYABLE` |
| Liveliness | tokens, liveliness subscriber (with history), liveliness `get` | `Z_FEATURE_LIVELINESS` |
| Matching | `z_publisher_get_matching_status`, `z_publisher_declare_matching_listener`, the same for queriers | `Z_FEATURE_MATCHING` (needs `INTEREST`) |
| Scouting | `z_scout`, `zp_hello_locators` | `Z_FEATURE_SCOUTING` (needs UDP unicast) |
| Timestamps | `z_timestamp_new` | — |
| Batching | `zp_batch_start`, `zp_batch_flush`, `zp_batch_stop` | `Z_FEATURE_BATCHING` |
| Cancellation | `z_cancellation_token_*` (for `get`) | unstable |
| Source info, entity IDs, reliability | `z_sample_source_info`, `z_publisher_id`, `z_sample_reliability` | unstable |
| Connectivity | `z_info_transports`, `z_info_links`, transport and link event listeners | `Z_FEATURE_CONNECTIVITY` (unstable) |
| Advanced pub/sub | `ze_declare_advanced_publisher`, `ze_declare_advanced_subscriber` (cache, history, recovery, miss detection) | `Z_FEATURE_ADVANCED_PUBLICATION` / `_SUBSCRIPTION` (unstable) |
| Admin space | `zp_start_admin_space`, `zp_stop_admin_space` | `Z_FEATURE_ADMIN_SPACE` (unstable, needs `QUERYABLE`) |
| Executor | `zp_spin_once` (single-thread builds) | `Z_FEATURE_MULTI_THREAD=0` |

What **isn't** there, compared with Rust Zenoh, is covered in [Limitations](limitations.md): routing,
router mode, authentication, ACL and interceptors, SHM, priorities on the transport, compression, storages,
plugins, and so on.

## Feature flags

Pass flags to CMake as `-DZ_FEATURE_X=0|1`. They end up as `#define`s in the generated
`include/zenoh-pico/config.h`. CMake's cache remembers them, so delete the build directory or override them
explicitly when you change them.

| Flag | Default | Effect | Requires / forced |
|---|---|---|---|
| `Z_FEATURE_UNSTABLE_API` | 0 | Compiles unstable API functions | — |
| `Z_FEATURE_MULTI_THREAD` | 1 | Background executor thread, mutexes. 0 = single thread, drive with `zp_spin_once` | Platform threads ([Platforms](platforms.md)) |
| `Z_FEATURE_PUBLICATION` | 1 | Publishing API | — |
| `Z_FEATURE_SUBSCRIPTION` | 1 | Subscribing API | — |
| `Z_FEATURE_QUERY` | 1 | `get` / querier | — |
| `Z_FEATURE_QUERYABLE` | 1 | Queryables | — |
| `Z_FEATURE_LIVELINESS` | 1 | Liveliness | — |
| `Z_FEATURE_INTEREST` | 1 | Interest protocol: write filtering and declarations on demand | — |
| `Z_FEATURE_MATCHING` | 1 | Matching status/listeners | Forced to 0 without `INTEREST` |
| `Z_FEATURE_SCOUTING` | 1 | `z_scout` and scouting at open | Forced to 0 without `LINK_UDP_UNICAST` |
| `Z_FEATURE_FRAGMENTATION` | 1 | Send and receive fragmented messages, patch-level fragment markers | — |
| `Z_FEATURE_BATCHING` | 1 | `zp_batch_*` | — |
| `Z_FEATURE_BATCH_TX_MUTEX` | 0 | Hold the TX lock for the whole batch (faster; keep-alives may stall) | — |
| `Z_FEATURE_BATCH_PEER_MUTEX` | 0 | Hold the peer lock for the whole batch (blocks reception while batching) | — |
| `Z_FEATURE_ENCODING_VALUES` | 1 | Predefined encoding constants | — |
| `Z_FEATURE_SESSION_CHECK` | 1 | Entities check that their session still exists | — |
| `Z_FEATURE_LOCAL_SUBSCRIBER` | 0 | Local publications reach local subscribers | — |
| `Z_FEATURE_LOCAL_QUERYABLE` | 0 | Local queries reach local queryables | — |
| `Z_FEATURE_RX_CACHE` | 0 | LRU cache (10 entries) for RX key-expression matching | — |
| `Z_FEATURE_UNICAST_TRANSPORT` | 1 | Unicast transports | — |
| `Z_FEATURE_MULTICAST_TRANSPORT` | 1 | Multicast transports | — |
| `Z_FEATURE_RAWETH_TRANSPORT` | 0 | Raw Ethernet transport | Linux |
| `Z_FEATURE_UNICAST_PEER` | 1 | Peer mode over unicast | — |
| `Z_FEATURE_AUTO_RECONNECT` | 1 | Reconnect after the transport is lost | — |
| `Z_FEATURE_MULTICAST_DECLARATIONS` | 0 | Declare key expressions on multicast (enables write filtering there, but every node re-sends all declarations whenever a node joins) | — |
| `Z_FEATURE_LINK_TCP` | 1 | `tcp/` | — |
| `Z_FEATURE_LINK_UDP_UNICAST` | 1 | `udp/` unicast | — |
| `Z_FEATURE_LINK_UDP_MULTICAST` | 1 | `udp/` multicast | Forced to 0 on FreeRTOS-Plus-TCP |
| `Z_FEATURE_LINK_SERIAL` | 0 | `serial/` | — |
| `Z_FEATURE_LINK_SERIAL_USB` | 0 | `serial/usb` (Raspberry Pi Pico) | Forced to 0 without `UNSTABLE_API` |
| `Z_FEATURE_LINK_TLS` | 0 | `tls/` | mbedtls 2.x/3.x via pkg-config |
| `Z_FEATURE_LINK_WS` | 0 | `ws/` | **Configure error** unless the platform is `emscripten` |
| `Z_FEATURE_LINK_BLUETOOTH` | 0 | `bt/` | Arduino-ESP32 |
| `Z_FEATURE_TCP_NODELAY` | 1 | Set `TCP_NODELAY` | — |
| `Z_FEATURE_CONNECTIVITY` | 0 | Transport/link info and events | Forced to 0 without `UNSTABLE_API` |
| `Z_FEATURE_ADVANCED_PUBLICATION` | 0 | Advanced publisher | Forced to 0 without `UNSTABLE_API`, `PUBLICATION` and `LIVELINESS` |
| `Z_FEATURE_ADVANCED_SUBSCRIPTION` | 0 | Advanced subscriber | Uses the unstable API |
| `Z_FEATURE_ADMIN_SPACE` | 0 | `@/<zid>/pico/**` admin space | Forced to 0 without `UNSTABLE_API` or `QUERYABLE` |

CMake prints the resolved set at configure time (`Building with feature config: …`).

!!! warning "Not every combination compiles"
    The non-release build types add `-Werror` (see [Platforms](platforms.md#linux-macos-bsd-windows)).
    Turning off many features at once can leave variables unused and stop the build. For example, a
    publish-only set (`MULTI_THREAD`, `SUBSCRIPTION`, `QUERY`, `QUERYABLE`, `LIVELINESS`, `INTEREST`,
    `MATCHING`, `SCOUTING`, both UDP links, `MULTICAST_TRANSPORT`, `UNICAST_PEER`, `ENCODING_VALUES`,
    `FRAGMENTATION`, `AUTO_RECONNECT` all set to 0) failed with `unused variable` errors in `src/net/session.c`
    and `src/session/loopback.c`. Each of those flags on its own built fine. Use `Release`, or add
    `-Wno-error`, for aggressive trimming.

## Footprint

Feature flags are how you trade features for flash and RAM. As a rough **relative** indication only (not
microcontroller numbers), here are the total `.text` sizes of the static library, built for Linux x86-64 with
GCC 13 at `-O3` (`Release`):

| Configuration | `.text` of `libzenohpico.a` |
|---|---|
| Defaults | ≈ 381 KB |
| Publish-only client, single thread (the set above) | ≈ 260 KB |

A real firmware image is much smaller than the archive total, because the linker drops unused code, and ARM
Thumb code is denser than x86-64. Measure with your own toolchain and `--gc-sections`.

RAM is mostly the per-transport buffers ([architecture](architecture.md#memory-and-ownership)): one 2048-byte
TX batch, one 2048-byte RX batch, defragmentation buffers growing up to 4096 bytes, plus the entity and
resource tables. The README's STM32/ThreadX recipe asks for a byte pool **larger than 25 kB**.

## Sources

- `zenoh-pico@1.10.1`: `CMakeLists.txt` (defaults, forced dependencies, summary), `include/zenoh-pico/config.h.in`,
  `docs/config.rst`, `include/zenoh-pico/api/*.h`
- Size measurements and the failing feature combination: built locally from the 1.10.1 tag
