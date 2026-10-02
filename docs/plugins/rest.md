# REST plugin

The REST plugin (`zenoh_plugin_rest`) maps HTTP requests to Zenoh operations. It's the easiest way to
reach Zenoh from scripts, browsers and monitoring tools.

## Configuration

```json5
plugins: {
  rest: {
    http_port: 8000,               // or "127.0.0.1:8000" / "[::1]:8000"
    work_thread_num: 2,            // tokio workers (only when loaded as a dynamic library)
    max_block_thread_num: 50,      // tokio blocking threads (same)
  },
},
```

| Key | Default | Meaning |
|---|---|---|
| `http_port` | **required** | A port number (binds `[::]:<port>`) or an `ip:port` string |
| `work_thread_num` | 2 | Worker threads of the plugin's own tokio runtime |
| `max_block_thread_num` | 50 | Blocking threads of that runtime |

`zenohd --rest-http-port <port|ip:port|none>` sets `http_port` and marks the plugin as required.

!!! note "IPv6-less hosts"
    A bare port number binds `[::]`. On hosts or containers without IPv6, startup fails with
    `Address family not supported by protocol`. Use `"0.0.0.0:8000"` there. (We hit this while writing these docs.)

Changing the REST config at runtime is **rejected** (`Runtime configuration change not supported`). Remove
and re-add `plugins/rest` instead.

## HTTP → Zenoh mapping

The URL path is the key expression: `http://host:8000/<key-expr>?<parameters>`.

| HTTP | Zenoh | Notes |
|---|---|---|
| `GET /<ke>` | `session.get(<ke>?<params>)` | Returns every reply. Consolidation is `Latest` unless the parameters contain a time range (`_time=…`), in which case it's `None`. A request body becomes the query payload, with `Content-Type` as its encoding |
| `GET /<ke>` with `Accept: text/event-stream` | `declare_subscriber(<ke>)` | **Server-Sent Events**: one event per sample, `event: PUT` or `event: DELETE`, `data:` holding the JSON sample |
| `PUT`, `POST` or `PATCH /<ke>` | `session.put(<ke>, body)` | `Content-Type` → Zenoh encoding |
| `DELETE /<ke>` | `session.delete(<ke>)` | |

`/@/local/...` is rewritten to `/@/<this router's zid>/...`, which is handy for the admin space.

CORS is open: `Access-Control-Allow-Origin: *`, methods GET/POST/PUT/PATCH/DELETE.

### Response formats for `GET`

Chosen in this order:

1. `Accept: text/event-stream` → SSE subscription.
2. `?_raw=true` (any value) in the query string → the **first** reply only, as raw bytes, with `Content-Type`
   from its encoding. A `;content-encoding=…` encoding suffix becomes a `Content-Encoding` header.
3. `Accept: text/html` → an HTML `<dl>` of key → value.
4. Otherwise → a JSON array:

```json
[
  { "key": "demo/a", "value": "hello", "encoding": "text/plain", "timestamp": "7692131887596940720/aaaa" }
]
```

How `value` is rendered:

| Sample encoding | `value` |
|---|---|
| `application/json`, `text/json`, `text/json5` | The parsed JSON (base64 string if parsing fails) |
| `text/plain`, `zenoh/string` | A string (base64 if it isn't valid UTF-8) |
| anything else | A **base64** string |
| empty payload | `null` |

Error replies appear with `"key": "ERROR"`.

!!! tip "Send text/plain"
    `curl -d` defaults to `application/x-www-form-urlencoded`, so the value comes back base64-encoded. Send
    `-H 'content-type: text/plain'` (or JSON) to get readable values back.

## Examples (verified)

```bash
# Put and get
curl -X PUT -H 'content-type: text/plain' -d 'hello' http://localhost:8000/demo/a
curl 'http://localhost:8000/demo/**'
# → [{"key":"demo/a","value":"hello","encoding":"text/plain","timestamp":"…/aaaa"}]

# Subscribe with SSE
curl -N -H 'accept: text/event-stream' 'http://localhost:8000/demo/**'
# event: PUT
# data: {"key":"demo/sse","value":"sse-test","encoding":"text/plain","timestamp":"…/aaaa"}

# Admin space
curl http://localhost:8000/@/local/router
curl 'http://localhost:8000/@/local/router/subscriber/**'

# Prometheus metrics, raw and gzip-encoded (needs zenoh built with `stats`)
curl --compressed 'http://localhost:8000/@/local/router/metrics?_raw=true'

# Runtime config: add a storage (admin space must be writable)
curl -X PUT -H 'content-type: application/json' -d '{"key_expr":"demo3/**","volume":"memory"}' \
  http://localhost:8000/@/local/router/config/plugins/storage_manager/storages/demo3
```

Selector parameters use `;` (not `&`): `?_time=[now(-1h)..];compression=false`.

## Admin space

`@/<zid>/router/status/plugins/rest/version` and `…/rest/port` (the effective config).

## Sources

- `plugins/zenoh-plugin-rest/src/lib.rs` (routes, `Accept` handling, `JSONSample`, `@/local`, CORS)
- `plugins/zenoh-plugin-rest/src/config.rs`
