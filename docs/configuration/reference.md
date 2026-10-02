# Configuration reference

This page lists **every key** of the Zenoh configuration, taken from the `Config` struct in
`commons/zenoh-config/src/lib.rs` and the defaults in `defaults.rs`. It includes keys that the upstream
`DEFAULT_CONFIG.json5` leaves out.

How to read the tables:

- **Path**: the key with `/` separators, as used by `--cfg`, `insert_json5` and the admin space.
- **Default**: the value used when the key is absent. *R / P / C* means router / peer / client
  ([mode-dependent](mode-dependent-values.md)).
- **Runtime**: whether the key can be changed on a running session. Only `plugins/**` can. See
  [Dynamic changes](dynamic-changes.md).
- :material-alert-circle-outline: marks a key that isn't in the upstream `DEFAULT_CONFIG.json5`.

---

## Identity

| Path | Type | Default | Description |
|---|---|---|---|
| `id` | hex string (u128, lowercase, no leading zeros, ≤ 32 chars) | random | Zenoh ID of the runtime. **Must be unique** in the network. |
| `mode` | `"router"` \| `"peer"` \| `"client"` | `peer` (`router` in `zenohd`) | Node role. See [Concepts](../concepts/index.md#modes-whatami). |
| `metadata` | any JSON | `null` | Free-form data, published in the admin space at `@/<zid>/<mode>` under `metadata`. |
| `region_name` | string, 1–32 bytes UTF-8 | `null` | This node's **north** region name. South-side gateways match it with `region_names` filters. See [Regions](../topology/regions.md). |

## Gateway (regions)

| Path | Type | Default | Description |
|---|---|---|---|
| `gateway/south` | `"auto"` or array of subregions | `"auto"` | How remote nodes are sorted into south subregions. `auto` puts peers and clients south of routers, and clients south of peers. |
| `gateway/south[i]/filters` | array of filters, or absent | absent | Absent: matches every remote. `[]`: matches none. Otherwise matches if **any** filter matches. |
| `gateway/south[i]/filters[j]/modes` | WhatAmI matcher | any | Remote's mode |
| `gateway/south[i]/filters[j]/interfaces` | non-empty list of interface names | any | Local interface the link arrived on |
| `gateway/south[i]/filters[j]/zids` | non-empty list of ZIDs | any | Remote's Zenoh ID |
| `gateway/south[i]/filters[j]/region_names` | non-empty list of region names | any | Remote's `region_name` |
| `gateway/south[i]/filters[j]/negated` :material-alert-circle-outline: | bool | `false` | Inverts the filter |

Within one filter **every** field must match (AND). See [Regions & gateways](../topology/regions.md).

## Connect

| Path | Type | Default | Description |
|---|---|---|---|
| `connect/endpoints` | mode-dependent list of endpoints or [groups](endpoints.md#endpoint-groups) | `[]` | Endpoints to connect to |
| `connect/timeout_ms` | mode-dependent i64 | R `-1`, P `-1`, C `0` | Overall connect timeout. `0` = one attempt, `-1` = no limit |
| `connect/exit_on_failure` | mode-dependent bool | R `false`, P `false`, C `true` | Fail session open if connecting fails |
| `connect/retry/period_init_ms` | mode-dependent i64 | `1000` | First retry delay |
| `connect/retry/period_max_ms` | mode-dependent i64 | `4000` | Upper bound on the retry delay |
| `connect/retry/period_increase_factor` | mode-dependent f64 | `2` | Backoff multiplier |

## Listen

| Path | Type | Default | Description |
|---|---|---|---|
| `listen/endpoints` | mode-dependent list of endpoints | R `["tcp/[::]:7447"]`, P `["tcp/[::]:0"]`, C none (empty if built without `transport_tcp`) | Endpoints to listen on |
| `listen/timeout_ms` | mode-dependent i64 | `0` | Overall listen timeout |
| `listen/exit_on_failure` | mode-dependent bool | `true` | Fail session open if a listener can't be bound |
| `listen/retry/*` | as `connect/retry/*` | as `connect/retry/*` | Listener retry backoff |

Details: [Endpoints & locators](endpoints.md).

## Open

| Path | Type | Default | Description |
|---|---|---|---|
| `open/return_conditions/connect_scouted` | bool | `true` | Session open waits (up to `scouting/delay`) for connections to scouted peers and routers. With `false`, the first publications and queries may be lost. |
| `open/return_conditions/declares` | bool | `true` | Session open waits for the initial declarations from connected peers. With `false`, startup may cause extra traffic. |

## Scouting

| Path | Type | Default | Description |
|---|---|---|---|
| `scouting/timeout` | u64 ms | `3000` | Client: how long to scout for a node to connect to before failing |
| `scouting/delay` | u64 ms | `500` | Peer and router: how long session open waits for scouting before returning |
| `scouting/multicast/enabled` | bool | `true` | Enable UDP multicast scouting |
| `scouting/multicast/address` | `ip:port` | `224.0.0.224:7446` | Multicast group used for scouting |
| `scouting/multicast/interface` | `"auto"`, or comma-separated interface names or IPs | `"auto"` | Interfaces to scout on. `auto` = every up, non-loopback, multicast-capable interface |
| `scouting/multicast/ttl` | u32 | `1` | Multicast TTL. On IPv6 groups, values > 1 log a warning |
| `scouting/multicast/autoconnect` | mode-dependent matcher | R `[]`, P/C `["router","peer","client"]` | Which kinds of scouted nodes to connect to |
| `scouting/multicast/autoconnect_strategy` | mode/target-dependent `"always"` \| `"greater-zid"` | `always` | Avoids duplicate connection attempts |
| `scouting/multicast/listen` | mode-dependent bool | `true` | Answer scout messages |
| `scouting/gossip/enabled` | bool | `true` | Enable gossip scouting |
| `scouting/gossip/multihop` | bool | `false` | Forward gossip beyond the next hop |
| `scouting/gossip/target` | mode-dependent matcher | R/P `["router","peer"]`, C `[]` | Which kinds of nodes to send gossip to |
| `scouting/gossip/autoconnect` | mode-dependent matcher | R `[]`, P/C `["router","peer","client"]` | Which kinds of gossiped nodes to connect to |
| `scouting/gossip/autoconnect_strategy` | mode/target-dependent | `always` | As for multicast |

Details: [Scouting](../discovery/scouting.md).

## Timestamping & queries

| Path | Type | Default | Description |
|---|---|---|---|
| `timestamping/enabled` | mode-dependent bool | R `true`, P `false`, C `false` | Add an HLC timestamp to data that has none |
| `timestamping/drop_future_timestamp` | bool | `false` | Drop messages timestamped in the future (instead of re-timestamping them) |
| `queries_default_timeout` | u64 ms | `10000` | Default timeout of `get` and queriers |

## Routing

| Path | Type | Default | Description |
|---|---|---|---|
| `routing/router/linkstate/transport_weights` | list of `{dst_zid, weight}` | `[]` | Link weights in router link-state routing (`weight` is a non-zero u16). Unset links weigh 100; when both ends set a weight, the larger one is used |
| `routing/interests/timeout` | u64 ms | `10000` | How long to wait for replies to interest declarations. If it expires, discovery may be incomplete |
| `routing/router/peers_failover_brokering` | — | — | **Deprecated, no effect** (logs a warning) |
| `routing/peer/*` | — | — | **Deprecated, no effect** (logs a warning) |

Details: [Routing](../topology/routing.md), [Interests](../discovery/interests.md).

## Aggregation

| Path | Type | Default | Description |
|---|---|---|---|
| `aggregation/subscribers` | list of key expressions | `[]` | Local subscribers on keys **included** in one of these are announced as a single subscriber on that expression |
| `aggregation/publishers` | list of key expressions | `[]` | Same for publishers |

Details: [Namespace, aggregation, timestamping](session-behaviour.md#aggregation).

## QoS

| Path | Type | Default | Description |
|---|---|---|---|
| `qos/publication` | list of `{key_exprs, config}` | `[]` | Overrides **publisher** QoS (congestion_control, priority, express, :material-flask: reliability, :material-flask: allowed_destination) for publishers on matching keys |
| `qos/network` | list of overwrite items | `[]` | Rewrites QoS of messages as they cross transports |

Details: [QoS overwrite](qos-overwrite.md).

## Transport: unicast

| Path | Type | Default | Description |
|---|---|---|---|
| `transport/unicast/open_timeout` | u64 ms | `10000` | Timeout to open a link (handshake included) |
| `transport/unicast/accept_timeout` | u64 ms | `10000` | Timeout to accept an incoming link |
| `transport/unicast/accept_pending` | usize | `100` | Maximum links in the handshake phase at once |
| `transport/unicast/max_sessions` | usize | `1000` | Maximum concurrent unicast transports |
| `transport/unicast/max_links` | usize | `1` | Maximum links per transport. > 1 needs `transport_multilink` |
| `transport/unicast/lowlatency` | bool | `false` | Use the low-latency transport. **Requires `qos/enabled: false`**. No fragmentation |
| `transport/unicast/qos/enabled` | bool | `true` | Per-priority queues and priority-aware transmission |
| `transport/unicast/compression/enabled` | bool | `false` | Compress batches (needs `transport_compression`, and both sides must agree) |

## Transport: multicast

| Path | Type | Default | Description |
|---|---|---|---|
| `transport/multicast/join_interval` | u64 ms | `2500` | Interval between JOIN messages |
| `transport/multicast/max_sessions` | usize | `1000` | Maximum remote peers per multicast group |
| `transport/multicast/qos/enabled` | bool | `false` | QoS on multicast. Off for zenoh-pico compatibility |
| `transport/multicast/compression/enabled` | bool | `false` | Compression on multicast. Off for zenoh-pico compatibility |

!!! warning
    Multicast transports don't negotiate anything. Every node in a group must use the **same** `batch_size`,
    QoS, compression and sequence-number settings.

## Transport: link

| Path | Type | Default | Description |
|---|---|---|---|
| `transport/link/protocols` | list of strings | all compiled in | Whitelist of protocols allowed for listening and connecting |
| `transport/link/tx/sequence_number_resolution` | `"8bit"` \| `"16bit"` \| `"32bit"` \| `"64bit"` | `32bit` | Sequence-number size. The smaller of the two sides is used |
| `transport/link/tx/lease` | u64 ms | `10000` | Lease announced to the remote. The link is closed if nothing is received for this long |
| `transport/link/tx/keep_alive` | usize | `4` | Keep-alives sent per lease period when idle |
| `transport/link/tx/batch_size` | u16 | `65535` | Maximum batch size (Zenoh's MTU). The effective value is the minimum of this, the remote's value and the link MTU |
| `transport/link/tx/threads` :material-alert-circle-outline: | usize | `1 + (num_cpus - 1) / 4` | Number of TX threads |
| `transport/link/tx/queue/size/{control,real_time,interactive_high,interactive_low,data_high,data,data_low,background}` | usize, **1–16** | `2` each | Batches per priority queue. Memory per queue = size × batch_size |
| `transport/link/tx/queue/congestion_control/drop/wait_before_drop` | i64 µs | `1000` | How long a `drop` message may wait for a free batch before it's dropped |
| `transport/link/tx/queue/congestion_control/drop/max_wait_before_drop_fragments` | i64 µs | `50000` | The same deadline for fragmented messages |
| `transport/link/tx/queue/congestion_control/block/wait_before_close` | i64 µs | `5000000` | How long a `block` message may wait before the **transport is closed** |
| `transport/link/tx/queue/batching/enabled` | bool | `true` | Adaptive batching under back-pressure |
| `transport/link/tx/queue/batching/time_limit` | u64 ms | `1` | Longest time a message is held back for batching |
| `transport/link/tx/queue/allocation/mode` | `"lazy"` \| `"init"` | `lazy` | Allocate queue batches on demand, or all up front |
| `transport/link/rx/buffer_size` | usize | `65535` | RX buffer per link. Also sizes the [io_uring](../transports/io-uring.md) buffer ring |
| `transport/link/rx/max_message_size` | usize | `1073741824` (1 GiB) | Largest message that will be reassembled from fragments. Larger ones are dropped |
| `transport/link/tcp/so_sndbuf` | u32 | OS default | TCP send buffer, for every `tcp/` link |
| `transport/link/tcp/so_rcvbuf` | u32 | OS default | TCP receive buffer, for every `tcp/` link |
| `transport/link/unixpipe/file_access_mask` :material-alert-circle-outline: | u32 | `0o777` | Permission mask for the named-pipe files created by `unixpipe/` listeners |

Details: [Transport layer](../transports/transport-layer.md), [Tuning](../transports/tuning.md).

### `transport/link/tls`

These apply to both `tls/` and `quic/` links. Per-endpoint `#` options override them (see [TLS](../transports/tls.md)).

| Path | Type | Default | Description |
|---|---|---|---|
| `root_ca_certificate` | path | none (WebPKI roots when connecting in router mode) | CA used to check the remote's certificate |
| `root_ca_certificate_base64` :material-alert-circle-outline: | base64 string (secret) | none | Same, inline |
| `listen_private_key` / `listen_private_key_base64` | path / base64 | none | Listener's private key |
| `listen_certificate` / `listen_certificate_base64` | path / base64 | none | Listener's certificate |
| `enable_mtls` | bool | `false` | Mutual TLS: listeners require client certificates |
| `connect_private_key` / `connect_private_key_base64` | path / base64 | none | Client key (mTLS) |
| `connect_certificate` / `connect_certificate_base64` | path / base64 | none | Client certificate (mTLS) |
| `verify_name_on_connect` | bool | `true` | Check the server certificate's name against the host being dialled |
| `close_link_on_expiration` | bool | `false` | Close links when the remote certificate chain expires (listeners need mTLS for this) |
| `so_sndbuf`, `so_rcvbuf` | u32 | OS default | TCP buffers for `tls/` links |

The `*_base64` fields are never serialized, so they don't show up in logs or in the admin space.

## Shared memory

Needs the `shared-memory` feature. Without it this section is accepted but does nothing.

| Path | Type | Default | Description |
|---|---|---|---|
| `transport/shared_memory/enabled` | bool | `true` | Advertise SHM support. Both sides must enable it |
| `transport/shared_memory/mode` | `"lazy"` \| `"init"` | `lazy` | Set up SHM on first use, or when the session opens |
| `transport/shared_memory/transport_optimization/enabled` | bool | `true` | Copy large regular payloads into SHM automatically |
| `transport/shared_memory/transport_optimization/pool_size` | non-zero usize bytes | `16777216` (16 MiB) | Pool used for that copying |
| `transport/shared_memory/transport_optimization/message_size_threshold` | usize bytes | `3072` | Minimum payload size that gets copied into SHM |
| `transport/shared_memory/transport_optimization/messages` | list of `put`, `delete`, `query`, `reply` | `["put","query","reply"]` | Message types eligible for automatic SHM |

Details: [SHM configuration](../shm/config.md).

## Authentication

| Path | Type | Default | Description |
|---|---|---|---|
| `transport/auth/usrpwd/user` | string | none | Username this node presents. `user` and `password` must be set together |
| `transport/auth/usrpwd/password` | string | none | Password this node presents |
| `transport/auth/usrpwd/dictionary_file` | path | none | File of accepted `user:password` lines |
| `transport/auth/pubkey/public_key_pem` | PEM string | none | RSA public key (inline) |
| `transport/auth/pubkey/private_key_pem` | PEM string | none | RSA private key (inline) |
| `transport/auth/pubkey/public_key_file` | path | none | RSA public key file |
| `transport/auth/pubkey/private_key_file` | path | none | RSA private key file |
| `transport/auth/pubkey/key_size` | usize bits | none | Key size |
| `transport/auth/pubkey/known_keys_file` | path | none | File of accepted public keys |

Details: [Authentication](../security/authentication.md).

## Admin space

| Path | Type | Default | Description |
|---|---|---|---|
| `adminspace/enabled` | bool | `false` (forced to `true` by `zenohd`) | Serve the `@/<zid>/<mode>/**` admin space |
| `adminspace/permissions/read` | bool | `true` | Answer admin-space queries |
| `adminspace/permissions/write` | bool | `false` | Accept `put`/`delete` on `@/<zid>/<mode>/config/**` |

Details: [Admin space](../admin-space/index.md).

## Namespace

| Path | Type | Default | Description |
|---|---|---|---|
| `namespace` | non-wildcard key expression | none | Prefix added to every outgoing key expression of the session and stripped from incoming ones |

## Interceptors

| Path | Type | Default | Description |
|---|---|---|---|
| `downsampling` | list of items | `[]` | Rate-limit messages per key expression. [Details](downsampling.md) |
| `low_pass_filter` | list of items | `[]` | Drop messages above a size limit. [Details](low-pass-filter.md) |
| `access_control` | object | disabled | ACL rules, subjects and policies. [Details](../security/access-control.md) |
| `access_control/enabled` | bool | `false` | |
| `access_control/default_permission` | `"allow"` \| `"deny"` | `deny` | |
| `access_control/rules`, `subjects`, `policies` | lists | none | |

## Statistics

| Path | Type | Default | Description |
|---|---|---|---|
| `stats/filters` | list of `{key: <keyexpr>}` | `[]` | Key expressions to keep per-key payload histograms for. Needs the `stats` feature. [Details](stats.md) |

## Plugins

| Path | Type | Default | Description |
|---|---|---|---|
| `plugins_loading/enabled` | bool | `false` (forced to `true` by `zenohd`) | Allow loading plugins from dynamic libraries |
| `plugins_loading/search_dirs` | list of paths or `{kind, value}` | `[{kind:"current_exe_parent"}, ".", "~/.zenoh/lib", "/opt/homebrew/lib", "/usr/local/lib", "/usr/lib"]` | Where to look for plugin libraries |
| `plugins/<name>` | object | — | One plugin's configuration. **Can be changed at runtime** |
| `plugins/<name>/__path__` | string or list of strings | — | Explicit library path(s). The first one that loads wins. Turns off the directory search |
| `plugins/<name>/__required__` | bool | `false` | Fail startup if the plugin can't be loaded |
| `plugins/<name>/__plugin__` :material-alert-circle-outline: | string | `<name>` | Library to load, when it differs from the config key. Lets you run two instances of one plugin |
| `plugins/<name>/__config__` | path | — | Merge another file into this object. [Details](index.md#including-other-files-__config__) |

Details: [Plugins](../plugins/index.md).

## Sources

- `commons/zenoh-config/src/lib.rs` (struct `Config`, validators)
- `commons/zenoh-config/src/defaults.rs`
- `commons/zenoh-config/src/gateway.rs`, `qos.rs`, `connection_retry.rs`, `mode_dependent.rs`
- `commons/zenoh-util/src/lib_search_dirs.rs`
- `DEFAULT_CONFIG.json5`
