# Dynamic changes

Can Zenoh's configuration be changed while it runs? **Only under `plugins/**`.** Every other key is read
when the session opens and stays fixed for the life of the session. This page shows the paths for making
changes, what reacts to them, and what to do for everything else.

## The rule, from the source

All runtime writes go through the runtime's config `Notifier` (`zenoh/src/api/config.rs`):

```rust
fn ensure_config_key_is_dynamically_writable(key: &str) -> ZResult<()> {
    if !key.starts_with("plugins/") {
        bail!("Error inserting conf value {} : updating config is only \
               supported for keys starting with `plugins/`", key);
    }
    Ok(())
}
```

`Notifier::insert_json5` and `Notifier::try_insert_json5_array_item` both call this check. `Config::remove`
has a matching restriction: *"Removal of values from Config is only supported for keys starting with
`plugins/`"*.

!!! note "Interceptors could be reloaded, but nothing calls it"
    `TablesLock::update_config` in `routing/dispatcher/tables.rs` can rebuild every interceptor (ACL,
    downsampling, QoS overwrite, low-pass) and the stats filters on live faces. In 1.10.1 it's marked
    `#[allow(dead_code)]` and only the tests call it. The same goes for `Runtime::update_network` (link-state
    weights). So hot-reloading these may arrive in a later release, but it isn't reachable today.

## Ways to change `plugins/**` at runtime

### 1. Through the admin space

Needs `adminspace/enabled: true` (always the case in `zenohd`) **and** `adminspace/permissions/write: true`
(`zenohd --adminspace-permissions rw`).

The admin space subscribes to `@/<zid>/<mode>/config/**`:

- **put** `@/<zid>/<mode>/config/<key>` with a UTF-8 JSON5 payload → `insert_json5(<key>, payload)`
- **delete** `@/<zid>/<mode>/config/<key>` → `remove(<key>)`
- If `<key>` ends in `<field>=<value>`, it's treated as an [array item by id](#array-items-by-id).

Failures (wrong permissions, non-plugin key, invalid JSON) are **only logged** on the target node. The
writer gets no error back:

```
ERROR Received PUT on '@/…/config/…' but adminspace.permissions.write=false in configuration
ERROR Error inserting conf value … : updating config is only supported for keys starting with `plugins/`
```

!!! tip "Wildcards apply to several nodes"
    A write is accepted when its key expression **intersects** the node's own `@/<zid>/<mode>/config/**`. A
    put on `@/*/router/config/plugins/...` therefore changes **every** router that allows writes. That's
    useful, and also easy to do by accident.

Example with the REST plugin:

```bash
# Add an in-memory storage to a running router
curl -X PUT \
  -H 'content-type: application/json' \
  -d '{"key_expr":"demo/new/**","volume":"memory"}' \
  http://localhost:8000/@/<zid>/router/config/plugins/storage_manager/storages/new

# Remove it
curl -X DELETE http://localhost:8000/@/<zid>/router/config/plugins/storage_manager/storages/new
```

Any Zenoh API can do the same: `session.put("@/<zid>/router/config/plugins/...", json)`.

### 2. From the application (Rust, `unstable`)

```rust
let config = session.config();                       // GenericConfig
let peers = config.get("connect/endpoints")?;        // JSON string (read works for any key)
config.insert_json5("plugins/rest/http_port", "8080")?; // only plugins/** is accepted
```

`GenericConfig` also has `get_typed::<T>(key)`, `get_plugin_config(name)` and `to_json()`.

### 3. At startup only: `--cfg`

`zenohd --cfg KEY:VALUE` uses the same key syntax but runs **before** the session opens, so any key can be
set that way. See [zenohd](../concepts/zenohd.md).

## What reacts to a change

When a `plugins/**` key changes (`zenohd`, or any build with `plugins` + `runtime_plugins`), the admin space
works out the difference between the requested and the running plugins:

| Change | Effect |
|---|---|
| New `plugins/<name>` object | Plugin `<name>` is loaded and started with that config |
| `plugins/<name>` removed | The plugin is **stopped** |
| `__path__` changed so it no longer includes the loaded library | The plugin is stopped and started again from the new path |
| Anything else inside `plugins/<name>/...` | Passed to the plugin's **config validator** (`ConfigValidator::check_config`) before it's stored. The plugin can reject it, accept it, or rewrite it, then react to it |

A plugin that doesn't override `config_checker` **rejects** every change to its settings with
`Runtime configuration change not supported`. Of the plugins in the main repo:

- **storage_manager** accepts changes: it works out the difference and adds or removes storages and volumes
  (`plugins/storage_manager/storages/<name>`, `.../volumes/<name>`). See
  [Storage manager](../plugins/storage-manager.md#runtime-changes).
- **rest** rejects in-place changes. To change its port, delete `plugins/rest` (which stops it) and then
  put a new `plugins/rest` object (which starts it again).

The validator only runs for plugins that have **started**. Settings for a plugin that isn't running are
stored without a check, and the plugin is started with them.

### Path syntax inside `plugins/`

Plugin settings are free-form JSON, and paths into them support array indexes:

- `plugins/x/list/0`: element 0. A path into `null` creates the array.
- `plugins/x/list/+`: **append** a new element.
- A numeric index past the end of the array is an error (`... not found`).

A write must leave `plugins/<name>` as a JSON **object**. Otherwise you get
`Attempt to provide non-object value as configuration for plugin`.

Changes are **not persisted**. They're lost when the process restarts. Update the config file as well.

## Observed behaviour

Captured from a `zenohd` 1.10.1 router (`id: "aaaa"`, admin space writable, REST on port 18000):

```bash
# 1. Add a storage at runtime: accepted, and the storage starts straight away
curl -X PUT -H 'content-type: application/json' -d '{"key_expr":"demo3/**","volume":"memory"}' \
  http://127.0.0.1:18000/@/aaaa/router/config/plugins/storage_manager/storages/demo3

# 2. Change a core key: rejected
curl -X PUT -d '1234' http://127.0.0.1:18000/@/aaaa/router/config/queries_default_timeout

# 3. Change a REST plugin setting: rejected by the plugin
curl -X PUT -d '8080' http://127.0.0.1:18000/@/aaaa/router/config/plugins/rest/http_port
```

Router log (the HTTP calls all returned success, because errors show up only here):

```text
WARN  zenoh::net::runtime::adminspace: Plugin `storage_manager` was already declared
WARN  zenoh::net::runtime::adminspace: Plugin `storage_manager` was already loaded from .../libzenoh_plugin_storage_manager.so
WARN  zenoh::net::runtime::adminspace: Plugin `storage_manager` was already started
ERROR zenoh::net::runtime::adminspace: Error inserting conf value @/aaaa/router/config/queries_default_timeout : 1234 - Error inserting conf value queries_default_timeout : updating config is only supported for keys starting with `plugins/`
ERROR zenoh::net::runtime::adminspace: Error inserting conf value @/aaaa/router/config/plugins/rest/http_port : 8080 - String("Runtime configuration change not supported ...")
```

The three `WARN` lines for case 1 are harmless. A change under `plugins/storage_manager/...` makes the
admin space re-check the already-running plugin before it hands the change to the plugin's validator. The
new storage then appears at `@/aaaa/router/status/plugins/storage_manager/storages/demo3`.

## Array items by id

Lists of objects that have an `id` field (`qos/network`, `downsampling`, `low_pass_filter`,
`access_control/rules`, …) can be addressed one item at a time:

```
<array-key>/<field>=<value>
```

- **Insert or update**: `try_insert_json5_array_item("qos/network/id=item1", "{ id: \"item1\", ... }")`.
  The object **must** contain `<field>: "<value>"`, otherwise you get `field filter mismatch`. It replaces
  the first matching element or is appended at the end.
- **Remove**: `try_remove_json5_array_item("qos/network/id=item1")` removes **every** element whose field matches.

This works on a `Config` before the session opens (for example in a binding that builds config in code)
and in the admin space key syntax. On a **running** session, only items under `plugins/**` can be changed
this way. A test (`runtime_try_insert_json5_array_item_rejects_non_plugin_keys`) checks that `qos/network`
is rejected at runtime.

## Changing anything else

For non-plugin settings (endpoints, ACL, QoS rules, transports, …), the options are:

1. **Restart** the process with the new config. Peers and clients reconnect on their own: clients cycle
   through their `connect/endpoints`, and peers and routers retry configured endpoints.
2. **Change topology from the outside**: put a new router in front, or rely on scouting to find new nodes.
3. **For application sessions**: close the session and open a new one with the new config.

## Sources

- `zenoh/src/api/config.rs` (`Notifier`, `ensure_config_key_is_dynamically_writable`)
- `commons/zenoh-config/src/lib.rs` (`remove`, `try_insert_json5_array_item`, `try_remove_json5_array_item`)
- `zenoh/src/net/runtime/adminspace.rs` (config subscriber, plugin diff, `ConfigValidator`)
- `zenoh/src/net/routing/dispatcher/tables.rs` (`update_config`, dead code)
