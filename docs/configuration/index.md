# Configuration

Every Zenoh runtime, whether a library session or `zenohd`, is driven by one configuration tree. This page
explains where the configuration comes from and how it's processed. The [full reference](reference.md)
lists every key.

## Where configuration comes from

| Source | API / CLI | Notes |
|---|---|---|
| Defaults | `Config::default()` | Empty tree. Each key falls back to the default in `defaults.rs` when read. |
| A file | `Config::from_file(path)`, `zenohd -c path` | Format is chosen by file extension (below) |
| An environment variable | `Config::from_env()` | Reads the **path** in `ZENOH_CONFIG` and loads that file |
| A JSON5 string | `Config::from_json5(str)`, `zenohd --cfg ':{...}'` | |
| Single keys | `config.insert_json5("key/path", "json5")`, `zenohd --cfg 'key/path:json5'` | Validated per key |
| Mode helpers | `zenoh::config::peer()`, `client(endpoints)` (Rust) | Preset `mode` (and `connect/endpoints` for client) |

Bindings expose the same functions, for example `zenoh.Config.from_file()` in Python and `z_config_from_file()` in C.
The [API section](../api/session.md#configuration) covers them.

### File formats

`Config::from_file` chooses the parser from the file extension:

| Extension | Parser | Notes |
|---|---|---|
| `.json5`, `.json` | JSON5 | Comments, unquoted keys and trailing commas are allowed |
| `.yaml`, `.yml` | YAML | |
| `.toml` | TOML | :material-flask: Only with the `unstable` feature, and logs a warning that the format may be removed |
| anything else | — | Error: `Unsupported file type` |
| no extension | — | Error: configuration files must have an extension |

An empty file is an error (`Empty config file`).

### Validation

The config is a `validated_struct`, and every section uses `deny_unknown_fields`. As a result:

- **A misspelt key is an error**, not something silently ignored. Most keys are optional and fall back to defaults when absent.
- Some fields have validators that run on load and on every `insert`:
    - `transport/link/tx/sequence_number_resolution` must be ≤ the maximum supported (`64bit`).
    - Each `transport/link/tx/queue/size/*` must be between **1 and 16**.
    - `transport/auth/usrpwd`: `user` and `password` must be set together or not at all.
- Lists marked *non-empty* in the reference (ACL `messages`, interceptor `interfaces`, …) reject `[]`.

`Config::from_json5` separates the two kinds of failure: JSON5 that doesn't parse, and JSON5 that parses but
fails validation ("The config was correctly deserialized, but it is invalid").

### Deprecated keys

These keys still parse, but log a warning and do nothing:

- `routing/router/peers_failover_brokering`
- `routing/peer` (including `routing/peer/mode` and `routing/peer/linkstate`)

## Mode-dependent values

Many keys take either a single value or a value per mode:

```json5
connect: { timeout_ms: -1 }                                   // all modes
connect: { timeout_ms: { router: -1, peer: -1, client: 0 } }  // per mode
```

See [Mode-dependent values](mode-dependent-values.md).

## Including other files (`__config__`)

Inside the `plugins` section, any object may contain `__config__: "path/to/file.json5"`. When the config
is loaded from a file:

- The file is loaded (JSON5, JSON or YAML) and its top-level keys are **merged into** the object. Keys in
  the included file win over keys in the parent object.
- Includes are processed recursively, so an included object may contain its own `__config__`.
- Paths are resolved relative to the current working directory for the top level, and relative to the
  including file's directory for nested includes.
- Include loops are detected and rejected.

!!! note
    `__config__` is only processed for the `plugins` tree, and only by `Config::from_file`. Core config
    sections can't be split across files this way.

## Secrets and `private`

- Any object key named `private` is **removed** when the config is printed (`Display`, logs) or served
  through the admin space. Plugins use this to hide credentials, for example `volumes/influxdb/private/password`.
- The TLS `*_base64` fields are held as secret values and never serialized.

## How the runtime uses the config

When a session opens, `Config::expanded()` fills in a random `id` if none is set, and sets `mode` to `peer`
if none is set (`zenohd` sets `router` before this). The runtime then keeps the config inside a `Notifier`.
You can read it back from the API (`session.config()` in Rust, with `unstable`). The admin space only
**accepts writes** under `@/<zid>/<mode>/config/**` and doesn't serve the config for reading. The startup
config is logged at INFO by `zenohd` (`Initial conf: ...`, with `private` keys removed).

Most settings are **read once at startup**. Only `plugins/**` can be changed while the session runs. See
[Dynamic changes](dynamic-changes.md).

## Sources

- `commons/zenoh-config/src/lib.rs` (`Config`, `from_file`, validators, `sift_privates`)
- `commons/zenoh-config/src/include.rs` (`__config__`)
- `commons/zenoh-config/src/defaults.rs`
- `zenoh/src/api/config.rs` (`ZENOH_CONFIG`, `from_json5`, `Notifier`)
