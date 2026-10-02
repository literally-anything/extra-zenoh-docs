# C++ (zenoh-cpp)

zenoh-cpp is a **header-only** C++17 wrapper with RAII types and lambdas. It can sit on either backend:

| Backend | CMake target | Notes |
|---|---|---|
| zenoh-c | `zenohcxx::zenohc` | Full feature set, SHM included when zenoh-c is built with SHM + unstable |
| zenoh-pico | `zenohcxx::zenohpico` | For embedded targets. Features follow pico's `Z_FEATURE_*` flags, no SHM |

```cmake
find_package(zenohc)        # and/or zenohpico
find_package(zenohcxx)
target_link_libraries(app PUBLIC zenohcxx::zenohc)
```

Unstable APIs need the backend built with its unstable flag (`Z_FEATURE_UNSTABLE_API`).

## API layout (`include/zenoh/api/`)

| Header | Contents |
|---|---|
| `session.hxx` | `Session::open`, `close`, `is_closed`, `get_zid`, `get_routers_z_id`, `get_peers_z_id`, `put`, `delete_resource`, `get`, `declare_publisher`, `declare_subscriber`, `declare_background_subscriber`, `declare_queryable`, `declare_querier`, `declare_keyexpr`, `liveliness_declare_token`, `liveliness_declare_subscriber`, `liveliness_get`, `new_timestamp`; 🔬 `get_transports`, `get_links`, `declare_transport_events_listener`, `declare_link_events_listener` |
| `publisher.hxx`, `subscriber.hxx`, `matching.hxx` | Pub/sub and matching listeners |
| `query.hxx`, `queryable.hxx`, `querier.hxx`, `reply.hxx`, `query_consolidation.hxx` | Query/reply |
| `liveliness.hxx`, `scout.hxx`, `hello.hxx` | Liveliness, scouting |
| `bytes.hxx`, `encoding.hxx`, `sample.hxx`, `timestamp.hxx`, `id.hxx`, `source_info.hxx` | Data model |
| `channels.hxx` | `channels::FifoChannel`, `channels::RingChannel` |
| `cancellation.hxx` | 🔬 `CancellationToken` |
| `transport*.hxx`, `link*.hxx` | 🔬 Connectivity |
| `ext/` | `serialization.hxx`, `advanced_publisher.hxx`, `advanced_subscriber.hxx`, `miss.hxx`, `publication_cache.hxx`, `querying_subscriber.hxx`, `session_ext.hxx` |
| `shm/` | 🔬 Providers, buffers, client storage, cleanup (zenoh-c backend only) |

## Conventions

- Errors are reported with exceptions (`zenoh::ZException`) by default. Most functions also take an optional
  `ZResult* err` out-parameter.
- Entities are move-only RAII objects. Destroying them undeclares.
- Callbacks take a `std::function`-like callable and an optional `on_drop` callable.

## Sources

- `zenoh-cpp@1.10.1`: `include/zenoh/api/`, `README.md`
