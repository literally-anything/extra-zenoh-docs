# Session

A `Session` is the entry point. Everything is declared on it, and closing it undeclares everything.

## Opening and closing

=== "Rust"

    ```rust
    let session = zenoh::open(zenoh::Config::default()).await?;
    // ...
    session.close().await?;
    ```

=== "Python"

    ```python
    import zenoh
    with zenoh.open(zenoh.Config()) as session:
        ...
    ```

=== "C"

    ```c
    z_owned_config_t config; z_config_default(&config);
    z_owned_session_t s;
    if (z_open(&s, z_move(config), NULL) < 0) { /* error */ }
    z_drop(z_move(s));   // closes
    ```

=== "TypeScript"

    ```ts
    const session = await Session.open(new Config("ws/127.0.0.1:10000"));
    await session.close();
    ```

- Opening a session starts a full Zenoh runtime (except in zenoh-ts, which connects to a remote one). It
  binds listeners, connects, scouts, and waits for the [start conditions](../configuration/reference.md#open).
- `close()` undeclares all entities and closes transports. In Rust, dropping the last handle also closes it,
  and `is_closed()` reports the state. Operations on a closed session return `SessionClosedError`.
- A session is cheap to clone (an `Arc`). Share one per process instead of opening many.
- Rust `close()` gives up after **10 s** with `close operation timed out!`. With `unstable`,
  `session.close().wait_callbacks()` also waits for callbacks that are still running. Entities hold the
  session through an internal `WeakSession` (exposed only with `internal`), so a forgotten subscriber
  doesn't keep the session alive. When the last `Session` handle is dropped, the session closes.

## Configuration

Every binding (except TypeScript) builds a config from defaults, a file, a JSON5 string or `ZENOH_CONFIG`,
and can set single keys:

| Operation | Rust | C | Python | Kotlin/Java | Go |
|---|---|---|---|---|---|
| Default | `Config::default()` | `z_config_default` | `Config()` | `Config.default()` (Kotlin), `Config.loadDefault()` (Java) | `NewConfigDefault()` |
| From file | `Config::from_file(p)` | `zc_config_from_file` | `Config.from_file(p)` | `Config.fromFile(p)` | `NewConfigFromFile(p)` |
| From JSON5 | `Config::from_json5(s)` | `zc_config_from_str` | `Config.from_json5(s)` | `Config.fromJson5(s)` (also `fromJson`, `fromYaml`) | `NewConfigFromStr(s)` |
| From env | `Config::from_env()` | `zc_config_from_env` | `Config.from_env()` | `Config.fromEnv()` | `NewConfigFromEnv()` |
| Set key | `config.insert_json5(k, v)` | `zc_config_insert_json5` | `config.insert_json5(k, v)` | `config.insertJson5(k, v)` | `config.InsertJson5(k, v)` |
| Get key | `config.get_json(k)` | `zc_config_get_from_str` | `config.get_json(k)` | `config.getJson(k)` | `config.Get(k)` |

zenoh-pico uses `zp_config_insert(config, Z_CONFIG_*_KEY, value)` with a small set of keys (mode,
connect, listen, multicast scouting, …). See [pico](languages/pico.md).

For **runtime** changes after open, see [Dynamic changes](../configuration/dynamic-changes.md): only
`plugins/**` can be changed, through `session.config()` (Rust, unstable) or the admin space.

## Declaring key expressions

`session.declare_keyexpr("robot/arm")` gives the expression a numeric ID that's shared with the network,
so later messages can send the short ID. Publishers do this for you automatically. Undeclare with
`session.undeclare(ke)`.

## Timestamps

`session.new_timestamp()` returns a `Timestamp` from the session's HLC, with the session's ZID as its ID.
Pass it to `put(...).timestamp(ts)` to timestamp data at the source (useful in peer-to-peer setups where
no router adds timestamps).

## Other session methods (Rust)

| Method | Purpose |
|---|---|
| `zid()` | This session's ZID |
| `info()` | [Session info](info.md) |
| `liveliness()` | [Liveliness](liveliness.md) |
| `config()` | :material-flask: Live config handle (`GenericConfig`) |
| `put`, `delete`, `get` | One-shot operations ([pubsub](pubsub.md), [query](query.md)) |
| `declare_publisher`, `declare_subscriber`, `declare_queryable`, `declare_querier` | Entities |

## Sources

- `zenoh/src/api/session.rs`, `config.rs`, `builders/session.rs`
- Binding sources listed on each [language page](index.md#languages)
