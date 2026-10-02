# Example configurations

Complete configuration files for common setups. **Every file on this page was loaded by `zenohd` 1.10.1**
(built with `zenoh/stats` and `shared-memory`), with host names, ports and certificate paths swapped for
local test values, and checked to start without configuration errors.

The files are also in the repository under [`examples/configs/`](https://github.com/literally-anything/extra-zenoh-docs/tree/main/examples/configs).

## 1. Production router

Plain router with a read-only admin space, REST bound to localhost, an in-memory storage, and multicast scouting answered but no auto-connect.

```json5 title="01-router.json5"
// Production router: TCP + TLS listeners, admin space read-only, REST on localhost,
// in-memory storage, timestamps on, multicast scouting answered but no autoconnect.
{
  mode: "router",
  metadata: { name: "router-1", site: "lab" },
  listen: { endpoints: ["tcp/0.0.0.0:7447"] },
  connect: { endpoints: [] },
  scouting: {
    multicast: { enabled: true, listen: true, autoconnect: { router: [] } },
    gossip: { enabled: true },
  },
  timestamping: { enabled: { router: true } },
  adminspace: { enabled: true, permissions: { read: true, write: false } },
  plugins_loading: { enabled: true },
  plugins: {
    rest: { http_port: "127.0.0.1:8000" },
    storage_manager: {
      storages: {
        cache: { key_expr: "app/**", volume: "memory", complete: false },
      },
    },
  },
}
```

## 2. Application peer

Finds other peers by multicast, also connects to a known router, and uses `greater-zid` between peers to avoid duplicate connections.

```json5 title="02-peer.json5"
// Application peer: finds others by multicast, also connects to a known router.
{
  mode: "peer",
  connect: { endpoints: ["tcp/10.0.0.1:7447"] },
  listen: { endpoints: ["tcp/0.0.0.0:0"] },
  scouting: {
    delay: 500,
    multicast: {
      enabled: true,
      interface: "auto",
      autoconnect: { peer: ["router", "peer"] },
      autoconnect_strategy: { peer: { to_router: "always", to_peer: "greater-zid" } },
    },
    gossip: { enabled: true, multihop: false },
  },
  open: { return_conditions: { connect_scouted: true, declares: true } },
}
```

## 3. Client with failover

Tries two routers in order and detects a lost connection within about 3 s (`lease: 3000`). With `timeout_ms: 0`, a client that can reach **no** router at startup fails to open (we observed `Unable to connect to any of [...]`), whatever `exit_on_failure` says. Use a negative `timeout_ms` to keep retrying instead.

```json5 title="03-client.json5"
// Client with failover between two routers and fast loss detection.
{
  mode: "client",
  connect: {
    endpoints: ["tcp/router-a.example.com:7447", "tcp/router-b.example.com:7447"],
    timeout_ms: { client: 0 },
    exit_on_failure: { client: false },
    retry: { period_init_ms: 500, period_max_ms: 8000, period_increase_factor: 2 },
  },
  scouting: { multicast: { enabled: false } },
  transport: { link: { tx: { lease: 3000, keep_alive: 4 } } },
}
```

## 4. TLS router with mTLS and per-CN ACL

Accepts only `tls/` with client certificates and authorises by certificate CN. Verified with clients holding `CN=sensor-1` and `CN=dashboard` (see [Access control](../security/access-control.md#verified-mtls-certificate-cn-subjects)). Certificates must be **X.509 v3**: a v1 certificate is rejected with `UnsupportedCertVersion`.

```json5 title="04-tls-mtls-acl.json5"
// Router accepting only TLS clients with certificates; per-CN permissions.
{
  mode: "router",
  listen: { endpoints: ["tls/0.0.0.0:7447"] },
  transport: {
    link: {
      protocols: ["tls"],
      tls: {
        root_ca_certificate: "/etc/zenoh/ca.pem",
        listen_private_key: "/etc/zenoh/router-key.pem",
        listen_certificate: "/etc/zenoh/router-cert.pem",
        enable_mtls: true,
        close_link_on_expiration: true,
      },
    },
  },
  scouting: { multicast: { enabled: false } },
  adminspace: { permissions: { read: true, write: false } },
  access_control: {
    enabled: true,
    default_permission: "deny",
    rules: [
      { id: "telemetry-pub", messages: ["put", "delete"], flows: ["ingress"], permission: "allow", key_exprs: ["telemetry/**"] },
      { id: "telemetry-sub", messages: ["declare_subscriber"], flows: ["ingress"], permission: "allow", key_exprs: ["telemetry/**"] },
      { id: "telemetry-out", messages: ["put", "delete"], flows: ["egress"], permission: "allow", key_exprs: ["telemetry/**"] },
    ],
    subjects: [
      { id: "sensors", cert_common_names: ["sensor-1", "sensor-2"] },
      { id: "dashboards", cert_common_names: ["dashboard"] },
    ],
    policies: [
      { rules: ["telemetry-pub"], subjects: ["sensors"] },
      { rules: ["telemetry-sub", "telemetry-out"], subjects: ["dashboards"] },
    ],
  },
}
```

## 5. Same-host shared-memory peer

Unix-socket listener and SHM set up at session open. Needs zenoh built with `shared-memory`.

```json5 title="05-shm-host.json5"
// Same-host peer using Unix sockets and shared memory (zenoh built with `shared-memory`).
{
  mode: "peer",
  listen: { endpoints: ["unixsock-stream//tmp/zenoh-app.sock"] },
  scouting: { multicast: { enabled: false } },
  transport: {
    shared_memory: {
      enabled: true,
      mode: "init",
      transport_optimization: { enabled: true, pool_size: 67108864, message_size_threshold: 4096, messages: ["put"] },
    },
  },
}
```

## 6. Edge router protecting a Wi-Fi uplink

Combines publication QoS, network QoS overwrite, downsampling and a low-pass filter on `wlan0` egress.

```json5 title="06-edge-gateway.json5"
// Edge router: protects a slow Wi-Fi uplink and a sensor network.
{
  mode: "router",
  listen: { endpoints: ["tcp/0.0.0.0:7447"] },
  qos: {
    publication: [ { key_exprs: ["robot/logs/**"], config: { priority: "background", congestion_control: "drop" } } ],
    network: [
      { id: "camera-to-bg", messages: ["put"], key_exprs: ["camera/**"], flows: ["egress"],
        interfaces: ["wlan0"], overwrite: { priority: "background", express: false } },
    ],
  },
  downsampling: [
    { id: "wifi-out", interfaces: ["wlan0"], flows: ["egress"], messages: ["put"],
      rules: [ { key_expr: "robot/lidar/**", freq: 5.0 }, { key_expr: "robot/imu", freq: 20.0 } ] },
  ],
  low_pass_filter: [
    { id: "no-big-on-wifi", interfaces: ["wlan0"], flows: ["egress"], messages: ["put", "reply"],
      key_exprs: ["**"], size_limit: 65536 },
  ],
}
```

## 7. Regions: HQ router

Places site routers in south subregions by `region_name`. See [Regions](../topology/regions.md).

```json5 title="07-regions-hq.json5"
// HQ router treating each site router as a south subregion.
{
  mode: "router",
  region_name: "hq",
  listen: { endpoints: ["tcp/0.0.0.0:7447"] },
  gateway: {
    south: [
      { filters: [ { region_names: ["site-a"] } ] },
      { filters: [ { region_names: ["site-b"] } ] },
    ],
  },
  routing: { router: { linkstate: { transport_weights: [] } } },
}
```

## Sources

- Validated against `zenohd` built from `eclipse-zenoh/zenoh@173b1220`.
