# Ecosystem plugins

These plugins live in their own repositories. Each one ships both as a `zenohd` plugin
(`libzenoh_plugin_<name>`) and, for the bridges, as a stand-alone `zenoh-bridge-<name>` executable that
embeds a Zenoh runtime. All were at **1.10.1** when this page was written. Match the plugin version
**exactly** to your `zenohd` ([compatibility](index.md#binary-compatibility)).

## ROS 2 ↔ Zenoh: `zenoh-plugin-ros2dds`

[Repository](https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds) · library `zenoh_plugin_ros2dds` · bridge `zenoh-bridge-ros2dds`

Bridges ROS 2 (over DDS) to Zenoh: topics, services and actions, with ROS 2 graph discovery. Key options
(`plugins/ros2dds`):

| Option | Purpose |
|---|---|
| `nodename`, `namespace`, `domain` | ROS identity of the bridge |
| `ros_localhost_only`, `ros_automatic_discovery_range`, `ros_static_peers` | DDS discovery scope |
| `shm_enabled` | Use DDS (Cyclone/iceoryx) shared memory |
| `allow` / `deny` | Regex lists per `publishers`, `subscribers`, `service_servers`, `service_clients`, `action_servers`, `action_clients` |
| `pub_max_frequencies` | Per-topic rate limit (`".*/laser_scan=5"`) |
| `pub_priorities` | Per-topic Zenoh priority and express (`"/scan=1:express"`) |
| `reliable_routes_blocking` | Use `block` congestion control for reliable topics |
| `queries_timeout` | Defaults and per-service/action timeouts |
| `transient_local_cache_multiplier` | History cache sizing for TRANSIENT_LOCAL topics |
| `work_thread_num`, `max_block_thread_num` | Runtime threads |

!!! note
    `rmw_zenoh` (the native ROS 2 Zenoh middleware) is a different project. It replaces DDS instead of bridging it.

## DDS ↔ Zenoh: `zenoh-plugin-dds`

[Repository](https://github.com/eclipse-zenoh/zenoh-plugin-dds) · library `zenoh_plugin_dds` · bridge `zenoh-bridge-dds`

Generic DDS bridge (it predates ros2dds). Options: `scope`, `domain`, `localhost_only`, `shm_enabled`,
`allow`/`deny` (topic regexes), `max_frequencies`, `generalise_subs`/`generalise_pubs`,
`forward_discovery`, `reliable_routes_blocking`, `queries_timeout`.

## MQTT ↔ Zenoh: `zenoh-plugin-mqtt`

[Repository](https://github.com/eclipse-zenoh/zenoh-plugin-mqtt) · library `zenoh_plugin_mqtt` · bridge `zenoh-bridge-mqtt`

An MQTT broker endpoint inside Zenoh. MQTT topics map to Zenoh keys. Options: `port` (default
`0.0.0.0:1883`), `scope` (key prefix), `allow`/`deny` (regex), `generalise_subs`/`generalise_pubs`,
`tx_channel_size`, `tls` (MQTTS), `auth.dictionary_file` (MQTT user/password).

## Web server

[Repository](https://github.com/eclipse-zenoh/zenoh-plugin-webserver) · library `zenoh_plugin_webserver`

An HTTP server that maps URLs to Zenoh keys and serves the results of a `get`, so you can host web content
from Zenoh storages (for example with the [filesystem backend](backends.md)). Requires `http_port`.

## Remote API (zenoh-ts)

[Repository](https://github.com/eclipse-zenoh/zenoh-ts) · plugin `zenoh-plugin-remote-api`

Exposes a WebSocket API that the **TypeScript binding** uses from browsers and Node.js. The browser talks
to this plugin, and the plugin runs the Zenoh operations inside `zenohd`. See [TypeScript](../api/languages/typescript.md).

## Others

The Eclipse Zenoh GitHub organisation hosts more experimental plugins and bridges. Check each repository's
README for its maintenance status and compatible Zenoh version.

## Sources

- `DEFAULT_CONFIG.json5` and `Cargo.toml` of each repository (cloned at their 1.10.1 heads)
- `zenohd/README.md` (list of known plugins)
