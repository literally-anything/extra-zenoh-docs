# API overview

Zenoh has one data model and one set of concepts, exposed through several language bindings. This section
covers **what APIs exist and what data you can use** in each language. It isn't a tutorial.

- Concept pages (this section) explain each API area using the Rust API as the reference, since every
  binding wraps or reimplements it.
- [Language pages](#languages) cover installation, naming conventions and the differences for each binding.

## Bindings at a glance

| Language | Package | Implementation | Talks to the network |
|---|---|---|---|
| [Rust](languages/rust.md) | `zenoh`, `zenoh-ext` (crates.io) | The reference implementation | Directly (full router/peer/client) |
| [C](languages/c.md) | `zenoh-c` | Rust core with a C ABI | Directly |
| [C++](languages/cpp.md) | `zenoh-cpp` (header-only) | Wraps **zenoh-c** or **zenoh-pico** | Directly |
| [Python](languages/python.md) | `eclipse-zenoh` (PyPI) | Rust core (PyO3) | Directly |
| [Kotlin](languages/kotlin.md) | `org.eclipse.zenoh:zenoh-kotlin` (+ `-android`) | Rust core (JNI) | Directly |
| [Java](languages/java.md) | `org.eclipse.zenoh:zenoh-java` (+ `-android`) | Rust core (JNI) | Directly |
| [TypeScript](languages/typescript.md) | `@eclipse-zenoh/zenoh-ts` (npm) | Remote client | **Through `zenohd` with the remote-api plugin** (WebSocket) |
| [zenoh-pico](languages/pico.md) | `zenoh-pico` | Separate C implementation for microcontrollers | Directly (client or peer) |
| [Go](languages/go.md) | `github.com/eclipse-zenoh/zenoh-go` | cgo over **zenoh-c** | Directly |

All bindings were checked at their **1.10.1** release tags.

## Capability matrix

✅ available · 🔬 available but unstable upstream (feature flag, annotation or "unstable" marker) ·
⚙️ off by default, enable at build time · ⚠️ partly available (see the cell) · ❌ not available

| Capability | Rust | C | C++ | Python | Kotlin | Java | TS | Pico | Go |
|---|---|---|---|---|---|---|---|---|---|
| Open/close session | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Config from file / JSON5 / env, `insert_json5` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ (locator only) | ⚠️ (`zp_config_insert` by key ID) | ✅ |
| Session info: own ZID, routers, peers | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Transports/links snapshot + events | 🔬 | 🔬 | 🔬 | 🔬 | ❌ | ❌ | ✅ | ⚙️🔬 | ✅ |
| Put / delete | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Publisher (QoS, encoding, attachment) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Subscriber (callback / channel) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Get / queryable / reply / reply_err / reply_del | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Querier | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Matching status / listener | ✅ | ✅ | ✅ | ✅ | ⚠️ (AdvancedPublisher only) | ❌ | ✅ | ✅ | ✅ |
| Liveliness (token, subscriber, get) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Scouting | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| New timestamp from session HLC | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Serialization (zenoh-ext format) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Advanced pub/sub (cache, history, miss detection) | 🔬 | 🔬 | 🔬 | 🔬 | 🔬 | ❌ | ❌ | ⚙️ | ✅ |
| Shared memory API | 🔬 | 🔬 | 🔬 (zenoh-c backend) | 🔬 | ❌ | ❌ | ❌ | ❌ | ❌ |
| Cancellation token | 🔬 | 🔬 | 🔬 | 🔬 | ❌ | ❌ | ✅ | 🔬 | ✅ |
| [Timestamp stack](timestamp-stack.md) (per-hop latency) | 🔬 | ❌ | ❌ | 🔬 | ❌ | ❌ | ❌ | ❌ | ❌ |
| [QueryingSubscriber / PublicationCache](zenoh-ext-legacy.md) (deprecated) | 🔬 | 🔬 | 🔬 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| [Group membership](zenoh-ext-legacy.md#group-membership-zenoh_extgroup) | 🔬 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

Notes:

- **Go** requires zenoh-c built with `ZENOHC_BUILD_WITH_UNSTABLE_API=ON`, so the unstable C APIs it wraps
  are always present.
- **C++** gets each feature from its backend. With the zenoh-pico backend, features follow pico's build flags,
  and SHM is unavailable.
- **zenoh-pico** turns features on with CMake flags (`Z_FEATURE_*`). Advanced pub/sub and the connectivity
  API are off by default. See [pico](languages/pico.md#build-features).
- **TypeScript** goes through `zenohd`'s remote-api plugin, so routing, transports and config are the
  router's. The client config only holds the plugin's WebSocket locator.

## Concept pages

| Page | Covers |
|---|---|
| [Data model](data-model.md) | `Sample`, `ZBytes`, `Encoding`, timestamps, QoS fields, attachments, source info |
| [Session](session.md) | Opening, closing, config handles, key expression declaration, timestamps |
| [Publish / subscribe](pubsub.md) | `put`, `delete`, `Publisher`, `Subscriber`, options |
| [Query / queryable / querier](query.md) | Selectors, parameters, targets, consolidation, replies |
| [Liveliness](liveliness.md) | Token, subscriber, get |
| [Handlers & channels](handlers.md) | Callbacks, FIFO/ring channels, background entities |
| [Session info & connectivity](info.md) | ZIDs, transports, links, events |
| [Scouting](scouting.md) | `scout()` and `Hello` |
| [Serialization](serialization.md) | The zenoh-ext wire format, shared by all languages |
| [Advanced pub/sub](advanced-pubsub.md) | `AdvancedPublisher` / `AdvancedSubscriber` |
| [Cancellation](cancellation.md) | Cancelling in-flight gets |
| [Timestamp stack](timestamp-stack.md) | Per-hop latency instrumentation |
| [zenoh-ext: Group, PublicationCache, QueryingSubscriber](zenoh-ext-legacy.md) | Lesser-known and deprecated zenoh-ext APIs |

## Languages

[Rust](languages/rust.md) · [C](languages/c.md) · [C++](languages/cpp.md) · [Python](languages/python.md) ·
[Kotlin](languages/kotlin.md) · [Java](languages/java.md) · [TypeScript](languages/typescript.md) ·
[zenoh-pico](languages/pico.md) · [Go](languages/go.md)

## Sources

- `eclipse-zenoh/zenoh@173b1220` (`zenoh/src/lib.rs`, `zenoh/src/api/`, `zenoh-ext/src/`)
- Binding repositories at tag `1.10.1` (`v1.10.1` for Go): `zenoh-c/include/zenoh_commons.h`,
  `zenoh-cpp/include/zenoh/api/`, `zenoh-python/zenoh/*.pyi`, `zenoh-kotlin/zenoh-kotlin/src/`,
  `zenoh-java/zenoh-java/src/`, `zenoh-ts/zenoh-ts/src/`, `zenoh-pico/include/zenoh-pico/api/`, `zenoh-go/zenoh/`
