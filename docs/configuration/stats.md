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
# Through the REST plugin (zenohd --rest-http-port 8000), without compression:
curl 'http://localhost:8000/@/*/router/metrics?compression=false'
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

Every metric is prefixed with `zenoh_` and carries the labels `local_id` (ZID) and `local_whatami`. The
OpenMetrics encoder adds the usual suffixes (`_total` for counters, `_bytes` for byte units, and
`_bucket`/`_sum`/`_count` for histograms).

| Family | Type | Extra labels | Meaning |
|---|---|---|---|
| `build` | info | `version` | Build version |
| `transports_opened` | gauge | — | Transports open now |
| `links_opened` | gauge family | protocol | Links open now |
| `resources_declared` | gauge family | `resource` (subscriber, queryable, token), `locality` (local, remote) | Declared entities |
| `tx_bytes`, `rx_bytes` | counter (bytes) | transport, link, `protocol` | Bytes on the wire |
| `tx_transport_message`, `rx_transport_message` | counter | transport, link, `protocol` | Transport-level messages (batches, keep-alives, …) |
| `tx_network_message`, `rx_network_message` | counter | transport, link, `priority`, `message`, `shm`, `protocol` | Network messages |
| `tx_network_message_payload`, `rx_network_message_payload` | histogram (bytes) | transport, `space` (user, admin), `priority`, `message`, `shm` | Payload sizes |
| `tx_network_message_dropped_payload`, `rx_network_message_dropped_payload` | histogram (bytes) | transport, `priority`, `message`, `protocol`, `reason` | Payloads dropped, by reason |
| `tx_network_message_payload_per_key`, `rx_network_message_payload_per_key` | histogram (bytes) | as payload, plus key | Only for keys in `stats/filters` |

Transport labels: `remote_zid`, `remote_whatami`, `remote_group` (multicast), `remote_cn` (TLS/QUIC
certificate CN). Link labels: `src_locator`, `dst_locator`.

Label values:

- `message`: `put`, `delete`, `query`, `reply`, `reply-err`, `response-final`, `interest`, `declare`, `oam`
- `reason`: `access-control`, `congestion`, `downsampling`, `low-pass`, `no-link`
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

Prometheus can't speak Zenoh. Expose the endpoint over HTTP with the [REST plugin](../plugins/rest.md) and
scrape that URL (`metrics_path: '/@/<zid>/router/metrics'`, `params: { compression: ['false'] }`), or put a
small exporter that queries `@/*/*/metrics` in front of it.

## Sources

- `commons/zenoh-stats/src/` (`registry.rs`, `labels.rs`, `histogram.rs`)
- `zenoh/src/net/runtime/adminspace.rs` (`metrics`, `_stats`)
- `zenoh/Cargo.toml` (`stats` feature)
