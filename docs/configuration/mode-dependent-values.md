# Mode-dependent values

One config file often serves routers, peers and clients alike. Many keys therefore accept a value that
depends on the node's `mode`.

## `ModeDependentValue<T>`

Two forms are accepted:

=== "Unique"

    ```json5
    timestamping: { enabled: true }   // applies whatever the mode is
    ```

=== "Per mode"

    ```json5
    timestamping: { enabled: { router: true, peer: false, client: false } }
    ```

In the per-mode form, any of `router`, `peer` and `client` may be left out. A missing entry falls back to
the built-in default **for that mode** from `defaults.rs`.

### Keys that accept mode-dependent values

| Key | Type | Built-in default |
|---|---|---|
| `connect/endpoints` | list of endpoints | `[]` |
| `connect/timeout_ms` | i64 | router `-1`, peer `-1`, client `0` |
| `connect/exit_on_failure` | bool | router `false`, peer `false`, client `true` |
| `connect/retry/period_init_ms` | i64 | `1000` |
| `connect/retry/period_max_ms` | i64 | `4000` |
| `connect/retry/period_increase_factor` | f64 | `2` |
| `listen/endpoints` | list of endpoints | router `["tcp/[::]:7447"]`, peer `["tcp/[::]:0"]`, client none |
| `listen/timeout_ms` | i64 | `0` |
| `listen/exit_on_failure` | bool | `true` |
| `listen/retry/*` | as for connect | as for connect |
| `scouting/multicast/autoconnect` | WhatAmI matcher | router `[]`, peer and client `["router","peer","client"]` |
| `scouting/multicast/autoconnect_strategy` | target-dependent (below) | `always` |
| `scouting/multicast/listen` | bool | `true` for all |
| `scouting/gossip/target` | WhatAmI matcher | router and peer `["router","peer"]`, client `[]` |
| `scouting/gossip/autoconnect` | WhatAmI matcher | router `[]`, peer and client `["router","peer","client"]` |
| `scouting/gossip/autoconnect_strategy` | target-dependent | `always` |
| `timestamping/enabled` | bool | router `true`, peer `false`, client `false` |

## WhatAmI matchers

A matcher is a set of modes. It's accepted as a list or as a `|`-separated string:

```json5
autoconnect: ["router", "peer"]
autoconnect: "router|peer"
autoconnect: []          // matches nothing
```

## `TargetDependentValue<T>` (autoconnect strategies)

`autoconnect_strategy` can depend on **this** node's mode *and* on the **remote** node's mode. Keys for
the remote's mode are prefixed with `to_`:

```json5
// one strategy for everything
autoconnect_strategy: "greater-zid"

// depends on the remote node's mode
autoconnect_strategy: { to_router: "always", to_peer: "greater-zid" }

// depends on this node's mode, then on the remote's
autoconnect_strategy: { peer: { to_router: "always", to_peer: "greater-zid" } }
```

When parsing, Zenoh first tries the per-mode form (`router`/`peer`/`client`) and then the target form.
See [Scouting](../discovery/scouting.md#autoconnect-strategies) for what the strategies do.

## Sources

- `commons/zenoh-config/src/mode_dependent.rs`
- `commons/zenoh-config/src/defaults.rs`
