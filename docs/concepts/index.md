# Concepts

This page covers the vocabulary the rest of the site uses.

## Session

A **session** is your application's handle on the Zenoh network. You get one by calling `zenoh::open(config)`.
Each session runs a full Zenoh **runtime**: it opens transports, takes part in discovery and routing, and
keeps the declarations of every entity the application created.

Each runtime has a **Zenoh ID** (ZID): an unsigned 128-bit integer, written as lowercase hex without leading
zeros. It's random unless you set `id` in the config. It must be unique in your network.

## Modes (`WhatAmI`)

Each runtime runs in one of three modes, set by the `mode` config key:

| Mode | Default for | What it does |
|---|---|---|
| `peer` | Library sessions (`zenoh::open`) | Finds other nodes (multicast scouting, gossip) and connects to them directly. Routes data for its own local entities. |
| `client` | — | Keeps a **single** connection to a router or peer that acts as its gateway. Suited to constrained devices. Takes no part in gossip. |
| `router` | `zenohd` | Runs a routing algorithm (link-state) over a topology you set up statically, and routes for peers and clients south of it. Doesn't auto-connect to what it scouts by default. |

`zenohd` sets `mode: "router"` when the config file has no `mode`. A library session left unset becomes
`peer` (`defaults::mode = Peer`). See [Topology](../topology/index.md) for how the modes combine and
[Regions](../topology/regions.md) for the newer north/south region model built on top of them.

## Entities

A session **declares** entities. Each one has an entity ID that is unique within the session, and
together they form an `EntityGlobalId` (ZID plus entity ID).

| Entity | Role | Paradigm |
|---|---|---|
| Publisher | Sends `put`/`delete` samples on a key expression | pub/sub |
| Subscriber | Receives samples matching a key expression | pub/sub |
| Queryable | Answers queries matching a key expression | query/reply |
| Querier | Sends repeated queries on a fixed key expression with preset options | query/reply |
| Liveliness token | Announces that something is alive while it's declared | liveliness |
| Liveliness subscriber | Is notified when matching tokens appear or disappear | liveliness |

You can also `put`, `delete` and `get` directly on the session without declaring anything.

Declarations travel through the network so routers and peers know where to send data. See
[Interests & declarations](../discovery/interests.md).

## Samples, payloads and metadata

Data arrives as a **Sample**: a key expression, a payload (`ZBytes`), a kind (`Put`/`Delete`), an encoding,
an optional timestamp, QoS (priority, congestion control, express, reliability), an optional attachment and,
with `unstable`, source info. The [data model](../api/data-model.md) page lists every field.

## QoS

Every message carries:

- **Priority**: 7 levels for data: `real_time` (1), `interactive_high` (2), `interactive_low` (3),
  `data_high` (4), `data` (5, default), `data_low` (6), `background` (7). Level 0 is `control`, which Zenoh
  reserves for itself.
- **Congestion control**: `drop` (default for put/delete), `block` (default for queries; a reply inherits
  its query's QoS unless the reply builder overrides it) or, with :material-flask: unstable, `block_first`.
  `block_first` blocks only for the first message sent with that strategy and drops the rest.
- **Express**: when `true`, the message skips batching and is sent straight away.
- **Reliability**: `reliable` or `best_effort`. Zenoh uses this to choose a link when there are several (see the [`rel`
  endpoint metadata](../configuration/endpoints.md#metadata)).

## Time

A router stamps data with a timestamp from a **Hybrid Logical Clock** (HLC) if it doesn't have one
yet (`timestamping/enabled`, on by default for routers only). Storages and advanced subscribers rely on these
timestamps. See [timestamping](../configuration/session-behaviour.md#timestamping).

## Where to go next

- [Key expressions](key-expressions.md): Zenoh's addressing language.
- [Cargo feature flags](feature-flags.md): what's compiled in.
- [zenohd](zenohd.md): the router binary.
