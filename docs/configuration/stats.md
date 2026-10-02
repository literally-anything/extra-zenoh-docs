# Statistics

With the `stats` Cargo feature, Zenoh records per-transport and per-link traffic statistics and serves them
through the admin space in **OpenMetrics** (Prometheus) text format.

!!! info "Needs a custom build"
    `stats` isn't a default feature, and stock `zenohd` builds don't include it. Build with
    `--features zenoh/stats` (for `zenohd`: `cargo build -p zenohd --features zenoh/stats`).
    Without it, the metrics endpoint returns only a `zenoh_build_info` line.

## Getting the metrics

Query `@/<zid>/<mode>/metrics`. The reply is OpenMetrics text with encoding
`application/openmetrics-text; version=1.0.0; charset=utf-8`, **gzip-compressed by default**.

```bash
# Through the REST plugin (zenohd --rest-http-port 8000), without compression.
# Zenoh selector parameters are separated by ';' (not '&'). The REST plugin returns
# this non-text payload base64-encoded inside its JSON reply.
curl -s 'http://localhost:8000/@/*/router/metrics?compression=false;descriptors=false' \
  | jq -r '.[0].value' | base64 -d
```

Selector parameters:

| Parameter | Default | Effect |
|---|---|---|
| `compression` | `true` | `false` returns plain text. Otherwise the payload is gzip and the encoding gets `;content-encoding=gzip` |
| `per_transport` | `true` | `false` aggregates instead of giving one series per remote |
| `per_link` | `true` | `false` aggregates the links of a transport |
| `disconnected` | `false` | `true` also reports stats kept from transports that have closed |
| `per_key` | `true` | `false` leaves out per-key histograms |
| `descriptors` | `true` | `false` removes the `# HELP`/`# TYPE` lines |

The JSON at the admin root (`@/<zid>/<mode>`) can include the same statistics if you add the `_stats`
parameter: `@/<zid>/router?_stats`.

## Metric families

Every metric is prefixed with `zenoh_` and carries `local_id` (ZID) and `local_whatami`. Counters and
histograms come in three variants: an **aggregate** (labelled by `protocol` only), a
**`_per_transport`** variant (adds the remote's labels) and a **`_per_link`** variant (also adds the
locators). The `per_transport` / `per_link` selector parameters turn the detailed variants on or off.

These are the `# TYPE` lines from a `zenohd` 1.10.1 built with `stats` (captured for this page):

| Family (as exposed) | Type | Notes |
|---|---|---|
| `zenoh_build` (`zenoh_build_info`) | info | `version` label |
| `zenoh_transports_opened` | gauge | Transports open now |
| `zenoh_links_opened` | gauge | Per `protocol` |
| `zenoh_{tx,rx}_bytes` | counter (`_total`) | Bytes on the wire. Also `_per_transport_bytes`, `_per_link_bytes` |
| `zenoh_{tx,rx}_transport_message` | counter | Transport-level messages (batches, keep-alives, …). Also `_per_transport`, `_per_link` |
| `zenoh_{tx,rx}_network_message` | counter | Network messages, labelled `priority`, `message`, `shm`. Also `_per_transport`, `_per_link` |
| `zenoh_{tx,rx}_network_message_payload_bytes` | histogram | Payload sizes, labelled `space` (user/admin), `priority`, `message`, `shm`. Also `_per_transport` |
| `zenoh_{tx,rx}_network_message_dropped_payload_bytes` | histogram | Dropped payloads, labelled `reason`. Also `_per_transport` |
| `zenoh_{tx,rx}_network_message_payload_per_key_bytes` | histogram | Only for keys in `stats/filters`. Also `_per_transport` |

Remote labels (per-transport/per-link series): `remote_zid`, `remote_whatami`, `remote_group` (multicast),
`remote_cn` (TLS/QUIC certificate CN), `disconnected`. Link labels: `src_locator`, `dst_locator`.

Example lines:

```text
zenoh_transports_opened{local_id="aaaa",local_whatami="router"} 1
zenoh_links_opened{local_id="aaaa",local_whatami="router",protocol="tcp"} 1
zenoh_tx_bytes_total{local_id="aaaa",local_whatami="router",protocol="tcp"} 255
zenoh_tx_per_transport_bytes_total{local_id="aaaa",local_whatami="router",protocol="tcp",remote_zid="bbbb",remote_whatami="router",remote_group="",remote_cn="",disconnected="false"} 255
zenoh_tx_network_message_total{local_id="aaaa",local_whatami="router",priority="data",message="put",shm="false",protocol="tcp"} 1
```

Label values:

- `message`: `put`, `delete`, `query`, `reply`, `reply-err`, `response-final`, `interest`, `declare`, `oam`
- `reason`: `access-control`, `congestion`, `downsampling`, `low-pass`, `no-link`
- `priority`: `control`, `real-time`, `interactive-high`, `interactive-low`, `data-high`, `data`, `data-low`, `background`
- histogram buckets (bytes): 0, 32, 1 Ki, 32 Ki, 1 Mi, 32 Mi, 1 Gi

## Per-key statistics

```json5
stats: {
  filters: [
    { key: "robot/camera/**" },
    { key: "robot/lidar/**" },
  ],
}
```

Each filter key expression gets its own payload histogram (`*_network_message_payload_per_key`), so you can
see which topics use the bandwidth. Keep the list short: each entry adds series to every transport.

## Scraping with Prometheus

The [REST plugin](../plugins/rest.md) can serve the metrics as a normal HTTP endpoint. With `?_raw=true` it
returns the first reply **as is**, with `Content-Type: application/openmetrics-text…` and
`Content-Encoding: gzip`. Prometheus handles both. `@/local` is a REST shortcut for the router's own ZID:

```bash
curl --compressed 'http://router:8000/@/local/router/metrics?_raw=true'
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: zenoh
    metrics_path: /@/local/router/metrics
    params: { _raw: ['true'] }
    static_configs:
      - targets: ['router:8000']
```

(Verified against `zenohd` 1.10.1 built with `zenoh/stats`.)

## Sources

- `commons/zenoh-stats/src/` (`registry.rs`, `labels.rs`, `histogram.rs`)
- `zenoh/src/net/runtime/adminspace.rs` (`metrics`, `_stats`)
- `zenoh/Cargo.toml` (`stats` feature)
