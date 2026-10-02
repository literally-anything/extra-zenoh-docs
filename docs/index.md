# Zenoh Extra Docs

In-depth documentation for [Eclipse Zenoh](https://github.com/eclipse-zenoh/zenoh), written from the source code.

The official site at [zenoh.io](https://zenoh.io) covers getting started well. It doesn't cover much of the
configuration, the transports, the newer features (regions, io_uring, interceptors, statistics) or how the
pieces fit together. This site fills those gaps.

!!! info "Version"
    All pages were checked against **zenoh 1.10.1**, commit
    [`173b1220`](https://github.com/eclipse-zenoh/zenoh/tree/173b1220c2ab59cc22c82bfc6c95ac9971ff213b),
    and the matching 1.10.1 releases of the language bindings. Most pages end with a **Sources** section
    listing the files the content was taken from, so you can check them against newer releases.

## What's here

| Section | What you'll find |
|---|---|
| [Architecture](architecture/index.md) | How the stack fits together (API → routing → transport → links), the wire protocol, threads, and a guide to adding a new transport |
| [Concepts](concepts/index.md) | Sessions, modes, entities, key expressions, Cargo features, the `zenohd` binary, environment variables and thread pools, the `z_*` example programs |
| [Configuration](configuration/index.md) | How config is loaded, **every** key with its default, endpoint syntax, interceptors (QoS overwrite, downsampling, low-pass), statistics, dynamic changes |
| [Transports](transports/index.md) | All 10 link protocols compared, with their options and limits; the transport layer (queues, batching, congestion control); tuning; io_uring |
| [Discovery](discovery/index.md) | Multicast and gossip scouting, interests, liveliness, matching, connectivity events |
| [Topology](topology/index.md) | Router/peer/client roles, routing, regions and gateways, deployment patterns |
| [Security](security/index.md) | User/password, public-key and TLS authentication; access control |
| [Shared memory](shm/index.md) | Zero-copy, the SHM API, and SHM tuning |
| [Admin space](admin-space/index.md) | Every `@/<zid>/...` key and what it returns |
| [Plugins](plugins/index.md) | Loading plugins, REST, storage manager, backends, ecosystem plugins, writing your own |
| [API](api/index.md) | What each language binding offers, plus the data model shared by all of them, the timestamp stack, and lesser-known zenoh-ext APIs |

## Conventions

- **Defaults** are taken from the code (`commons/zenoh-config/src/defaults.rs` and the `Config` struct),
  not from comments. Where the two disagree, the page says so.
- Badges used in tables:
    - :material-flask: **unstable**: only available with the `unstable` Cargo feature.
    - :material-puzzle: **feature `x`**: needs the Cargo feature `x` when building zenoh.
    - :material-linux: **Linux**: works on Linux only.
- Config key paths use `/` as separator (`transport/link/tx/batch_size`), the same form the
  `--cfg` flag, `Config::insert_json5` and the admin space use.
