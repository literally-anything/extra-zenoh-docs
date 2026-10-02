# C (zenoh-c)

zenoh-c exposes the Rust implementation through a C ABI. The headers are `zenoh.h` (which includes
`zenoh_commons.h`, `zenoh_concrete.h`, `zenoh_macros.h`, …).

## Build options (CMake)

| Option | Default | Effect |
|---|---|---|
| `ZENOHC_BUILD_WITH_UNSTABLE_API` | `OFF` | Defines `Z_FEATURE_UNSTABLE_API` and compiles the unstable functions (309 of the 879 API functions at 1.10.1) |
| `ZENOHC_BUILD_WITH_SHARED_MEMORY` | `OFF` | Defines `Z_FEATURE_SHARED_MEMORY`. The SHM API also needs `UNSTABLE_API` |
| `ZENOHC_CARGO_FLAGS` | — | Extra cargo flags, for example `--features=zenoh/transport_serial` |
| `ZENOHC_CARGO_CHANNEL`, `ZENOHC_CUSTOM_TARGET` | — | Toolchain and cross-compilation |

Packages: `.deb` and release archives from the zenoh-c GitHub releases, or build from source with CMake.

## Ownership model

zenoh-c uses Rust-like ownership with explicit types:

| Kind | Example | Meaning |
|---|---|---|
| `z_owned_X_t` | `z_owned_session_t` | You own it and must `z_drop(z_move(x))` it |
| `z_loaned_X_t` | `const z_loaned_session_t*` | Borrowed. Get one with `z_loan(x)` / `z_loan_mut(x)` |
| `z_moved_X_t` | `z_moved_config_t*` | Ownership is transferred into the call. Pass `z_move(x)` |
| `z_view_X_t` | `z_view_keyexpr_t` | Non-owning view of external memory (for example a string literal) |
| `z_X_t` | `z_timestamp_t`, `z_id_t` | Plain copyable values |

Generic macros (`z_loan`, `z_move`, `z_drop`, `z_clone`, `z_internal_check`, `z_internal_null`) dispatch
on the type (`_Generic` in C11, overloads in C++).

## Naming

| Prefix | Meaning |
|---|---|
| `z_` | Core Zenoh API (shared with zenoh-pico) |
| `zc_` | zenoh-c specific (config from file or JSON5, logging, SHM cleanup) |
| `ze_` | zenoh-ext (serialization, advanced pub/sub, querying subscriber) |

## Main entry points

| Area | Functions |
|---|---|
| Config | `z_config_default`, `zc_config_from_file`, `zc_config_from_str`, `zc_config_from_env`, `zc_config_insert_json5`, `zc_config_get_from_str`, `zc_config_to_string` |
| Session | `z_open`, `z_close`, `z_session_is_closed`, `z_info_zid`, `z_info_routers_zid`, `z_info_peers_zid`, `z_timestamp_new` |
| Pub/sub | `z_put`, `z_delete`, `z_declare_publisher`, `z_publisher_put`, `z_publisher_delete`, `z_declare_subscriber`, `z_declare_background_subscriber` |
| Matching | `z_publisher_get_matching_status`, `z_publisher_declare_matching_listener`, `z_querier_get_matching_status`, `z_querier_declare_matching_listener` |
| Query | `z_get`, `z_declare_queryable`, `z_query_reply`, `z_query_reply_err`, `z_query_reply_del`, `z_declare_querier`, `z_querier_get` |
| Liveliness | `z_liveliness_declare_token`, `z_liveliness_declare_subscriber`, `z_liveliness_get` |
| Scouting | `z_scout` |
| Key expressions | `z_keyexpr_from_str`, `z_keyexpr_canonize`, `z_keyexpr_intersects`, `z_keyexpr_includes`, `z_keyexpr_join`, `z_keyexpr_concat`, `z_declare_keyexpr` |
| Handlers | `z_closure_*`, `z_fifo_channel_*_new`, `z_ring_channel_*_new` |
| Serialization | `ze_serialize_*`, `ze_deserialize_*`, `ze_serializer_*`, `ze_deserializer_*` |
| 🔬 Connectivity | `z_info_transports`, `z_info_links`, `z_declare_transport_events_listener`, `z_declare_link_events_listener` |
| 🔬 Advanced | `ze_declare_advanced_publisher`, `ze_declare_advanced_subscriber`, `ze_advanced_subscriber_declare_sample_miss_listener`, `ze_declare_querying_subscriber` |
| 🔬 SHM | `z_posix_shm_provider_new`, `z_shm_provider_alloc*`, `z_alloc_layout_*`, `zc_cleanup_orphaned_shm_segments` |
| 🔬 Other | `z_cancellation_token_*`, `z_source_info_*`, `z_keyexpr_relation_to` |
| Logging | `zc_init_log_from_env_or`, `zc_try_init_log_from_env`, `zc_init_log_with_callback` |

Options structs follow the pattern `z_X_options_t` + `z_X_options_default(&opts)`, for example
`z_put_options_t` with `encoding`, `congestion_control`, `priority`, `is_express`, `timestamp`,
`attachment`, and (unstable) `reliability`, `allowed_destination`, `source_info`.

## Sources

- `zenoh-c@1.10.1`: `include/zenoh_commons.h`, `CMakeLists.txt`, `README.md`
