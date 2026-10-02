# Plugins

`zenohd` loads **plugins** to add features: an HTTP REST API, storages, bridges to MQTT/DDS/ROS 2, a web
server, and others. A plugin runs inside the router process and uses the router's own Zenoh runtime.

## Enabling plugin loading

```json5
plugins_loading: {
  enabled: true,                       // zenohd forces this to true
  search_dirs: [
    { kind: "current_exe_parent" },    // directory of the zenohd binary
    ".",                               // current working directory
    "~/.zenoh/lib",
    "/opt/homebrew/lib",
    "/usr/local/lib",
    "/usr/lib",
  ],                                   // the list above is the default
},
plugins: {
  rest: { http_port: 8000 },
  storage_manager: { storages: { demo: { key_expr: "demo/**", volume: "memory" } } },
},
```

- `search_dirs` entries are paths (strings or `{ kind: "path", value: "…" }`) or
  `{ kind: "current_exe_parent" }`. `zenohd --plugin-search-dir` **replaces** the list.
- Every key under `plugins` is a plugin to load. **Plugins are only loaded if they're in the config at
  startup** or added later through the [admin space](../configuration/dynamic-changes.md).

## Library lookup

For a plugin configured as `plugins/<name>`, `zenohd` looks for:

| Item | Value |
|---|---|
| Library name | `<prefix>zenoh_plugin_<name>.<suffix>`, for example `libzenoh_plugin_rest.so` (Linux), `libzenoh_plugin_rest.dylib` (macOS), `zenoh_plugin_rest.dll` (Windows) |
| Where | Each `search_dirs` entry, in order |
| `__path__` | A string or list of strings. Search is **off**, and the first path that loads is used |
| `__plugin__` | Load a different library name than the config key, for example two instances `rest_a` and `rest_b` with `__plugin__: "rest"` |
| `__required__` | `true` makes `zenohd` **fail** if the plugin can't be loaded or started (plugins should then also panic on fatal errors). Default `false`: log the error and continue |
| `__config__` | Path of a file merged into this plugin's config ([details](../configuration/index.md#including-other-files-__config__)) |

Storage backends follow the same scheme with the prefix `zenoh_backend_` (see [Backends](backends.md)).

## ⚠️ Binary compatibility

Plugins are Rust dynamic libraries loaded into `zenohd`. Before starting one, `zenohd` compares
(`zenoh-plugin-trait/src/compatibility.rs`):

1. the **rustc version** used to build each one;
2. the **Zenoh version**, including the git commit unless either side reports `release` (built from crates.io);
3. the **Zenoh feature set** (`zenoh::FEATURES`).

Any difference is refused:

```text
Incompatible rustc versions: host: … plugin: …
Incompatible Zenoh versions: host: … plugin: …
Incompatible Zenoh feature sets: host: … plugin: …
```

In practice: **build zenohd and its plugins with the same toolchain, the same Zenoh version and the same
features.** If you rebuild `zenohd` with `zenoh/stats` or `shared-memory`, rebuild the plugins with them too.
Building everything in one `cargo build` invocation guarantees this. The official release packages are built
to match each other.

## From the command line

```bash
zenohd -P rest                               # load libzenoh_plugin_rest from search dirs, required
zenohd -P myplugin:/opt/zenoh/libmy.so       # explicit path, required
zenohd --rest-http-port 8000                 # shortcut for the REST plugin
zenohd --plugin-search-dir /opt/zenoh/lib
```

## Status in the admin space

- `@/<zid>/router/plugins/**`: name, version, path, `state` (`Started`, …) and report for each plugin
  (backends included, for example `storage_manager/memory`).
- `@/<zid>/router/status/plugins/<name>/**`: data published by the plugin itself.

See the [Admin space reference](../admin-space/reference.md#plugins) and [status/plugins](../admin-space/reference.md#statusplugins).

## Plugins in the main repository

| Plugin | Library | Page |
|---|---|---|
| REST | `zenoh_plugin_rest` | [REST plugin](rest.md) |
| Storage manager | `zenoh_plugin_storage_manager` (includes the `memory` backend) | [Storage manager](storage-manager.md) |
| Example | `zenoh_plugin_example` | [Writing a plugin](writing-a-plugin.md) |

Separate repositories (all at 1.10.1): ROS 2/DDS, DDS, MQTT, web server, storage backends, and the
remote-api plugin used by zenoh-ts. See [Ecosystem plugins](ecosystem.md) and [Backends](backends.md).

## Sources

- `commons/zenoh-config/src/lib.rs` (`PluginsConfig`, `load_requests`, `PluginsLoading`)
- `commons/zenoh-util/src/lib_search_dirs.rs`
- `zenoh/src/api/loader.rs`, `zenoh/src/api/plugins.rs` (`PLUGIN_PREFIX`)
- `plugins/zenoh-plugin-trait/src/` (`compatibility.rs`, `manager/`)
- `zenohd/src/main.rs`
