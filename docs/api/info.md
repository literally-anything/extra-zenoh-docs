# Session info & connectivity

The full description, including every `Transport` and `Link` field and the event semantics, is on
[Discovery → Connectivity events](../discovery/connectivity-events.md). This page maps the API per language.

| Capability | Rust | C | Python | TypeScript | Go | Kotlin/Java |
|---|---|---|---|---|---|---|
| Own ZID | `session.zid()` / `info().zid()` | `z_info_zid` | `session.zid()` | `info().zid()` | `session.ZId()` | `info().zid()` |
| Router ZIDs | `info().routers_zid()` | `z_info_routers_zid` | `info.routers_zid()` | `info().routersZid()` | `RoutersZId()` | `info().routersZid()` |
| Peer ZIDs | `info().peers_zid()` | `z_info_peers_zid` | `info.peers_zid()` | `info().peersZid()` | `PeersZId()` | `info().peersZid()` |
| Transports | :material-flask: `info().transports()` | :material-flask: `z_info_transports` | `info.transports()` | `info().transports()` | `Transports()` | ❌ |
| Links | :material-flask: `info().links()` | :material-flask: `z_info_links` | `info.links()` | `info().links()` | `Links()` | ❌ |
| Transport events | :material-flask: `transport_events_listener()` | :material-flask: `z_declare_transport_events_listener` | `info.declare_transport_events_listener()` | `transportEventsListener()` | `DeclareTransportEventsListener` | ❌ |
| Link events | :material-flask: `link_events_listener()` | :material-flask: `z_declare_link_events_listener` | `info.declare_link_events_listener()` | `linkEventsListener()` | `DeclareLinkEventsListener` | ❌ |

C++ follows C (`Session::get_zid`, `get_routers_z_id`, `get_peers_z_id`, `get_transports`, `get_links`,
`declare_transport_events_listener`, …). zenoh-pico has `z_info_*` and, with
`Z_FEATURE_CONNECTIVITY=1` (off by default, unstable), transports, links and events.

Without the unstable API, a Rust application can still watch its own connectivity through the
[session-local admin space](../admin-space/reference.md#session-local-admin-space-zidsession)
(`@/<own zid>/session/transport/**`).

## Sources

- `zenoh/src/api/info.rs`, `connectivity.rs`, `admin.rs`
- Binding sources at 1.10.1
