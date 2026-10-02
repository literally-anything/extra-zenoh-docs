# Data model

Every binding exposes the same data. This page lists what a sample, a query and a reply carry, and the
types used for payloads and metadata. Names are the Rust ones. See the language pages for each binding's
spelling.

## Sample

What subscribers receive and what `get` replies contain (`zenoh::sample::Sample`):

| Field | Type | Meaning |
|---|---|---|
| `key_expr()` | `KeyExpr` | The concrete key of the data |
| `payload()` / `payload_mut()` | `ZBytes` | The data |
| `kind()` | `SampleKind` | `Put` or `Delete` |
| `encoding()` | `Encoding` | How to interpret the payload ([below](#encoding)) |
| `timestamp()` | `Option<Timestamp>` | HLC timestamp, if the publisher or a router added one ([Timestamping](../configuration/session-behaviour.md#timestamping)) |
| `congestion_control()` | `CongestionControl` | `Drop`, `Block` (`BlockFirst` with unstable) |
| `priority()` | `Priority` | `RealTime` … `Background` |
| `express()` | `bool` | Sent without batching |
| `reliability()` | `Reliability` | :material-flask: `Reliable` / `BestEffort` |
| `attachment()` / `attachment_mut()` | `Option<ZBytes>` | User metadata sent alongside the payload |
| `source_info()` | `Option<SourceInfo>` | :material-flask: Publisher's `EntityGlobalId` and sequence number |
| `timestamp_stack()` | `Option<TimestampStack>` | :material-flask: Per-hop timestamp instrumentation |

`SampleFields` is a version of the same struct with public fields, so you can destructure a sample
without cloning. `SampleBuilder` creates samples by hand, which is useful in tests.

## ZBytes

`ZBytes` is Zenoh's payload type, a possibly non-contiguous list of byte slices:

- It can be built from `Vec<u8>`, `&[u8]`, `String`/`&str`, `bytes::Bytes`, and SHM buffers.
- `to_bytes()` returns a `Cow<[u8]>` (it copies only if the data isn't contiguous). `try_to_string()` reads UTF-8.
- `reader()`/`writer()` give streaming access, and `slices()` iterates the underlying slices without copying.
- With `shared-memory` + `unstable`, `as_shm()`/`as_shm_mut()` expose SHM-backed payloads ([SHM API](../shm/api.md)).
- For typed data, use [zenoh-ext serialization](serialization.md), which works the same way in every language.

## Encoding

An `Encoding` is a **numeric ID** (sent compactly on the wire) plus an optional **schema** string
(for example `text/plain;charset=utf-8`, where `charset=utf-8` is the schema). Zenoh doesn't interpret it,
apart from the REST plugin's JSON rendering. Predefined encodings:

| Group | Encodings |
|---|---|
| Zenoh | `zenoh/bytes` (default), `zenoh/string`, `zenoh/serialized` (zenoh-ext format) |
| Generic | `application/octet-stream`, `text/plain` |
| Structured text | `application/json`, `text/json`, `text/json5`, `application/yaml`, `text/yaml`, `application/xml`, `text/xml`, `text/csv`, `text/html`, `text/css`, `text/javascript`, `text/markdown`, `application/x-www-form-urlencoded`, `application/sql`, `application/jsonpath`, `application/json-patch+json`, `application/json-seq`, `application/jwt`, `application/soap+xml`, `application/yang`, `application/openmetrics-text` |
| Binary serialisations | `application/cdr`, `application/cbor`, `application/protobuf`, `application/python-serialized-object`, `application/java-serialized-object`, `application/coap-payload` |
| Images | `image/png`, `image/jpeg`, `image/gif`, `image/bmp`, `image/webp` |
| Audio | `audio/aac`, `audio/flac`, `audio/mp4`, `audio/ogg`, `audio/vorbis` |
| Video | `video/h261`, `video/h263`, `video/h264`, `video/h265`, `video/h266`, `video/mp4`, `video/ogg`, `video/raw`, `video/vp8`, `video/vp9`, `application/mp4` |

Any other MIME string can be used too (it's sent as a string). Predefined ones cost less on the wire.

## Timestamp

`Timestamp` = **NTP64 time** + **ID** (by default the ZID of the node that created it, from its HLC).
Timestamps are totally ordered: by time, then by ID. Get one from the session with `session.new_timestamp()`
(see the [matrix](index.md#capability-matrix) for which bindings support it). In text form a timestamp looks like
`7692131887596940720/aaaa`, meaning `<ntp64>/<id>`.

## QoS types

| Type | Values |
|---|---|
| `Priority` | `RealTime` (1), `InteractiveHigh` (2), `InteractiveLow` (3), `DataHigh` (4), `Data` (5, default), `DataLow` (6), `Background` (7) |
| `CongestionControl` | `Drop` (default for put/delete), `Block` (default for queries), :material-flask: `BlockFirst` |
| `Reliability` | :material-flask: `Reliable` (default), `BestEffort`. A link-selection hint, not a retransmission guarantee |
| `Locality` | `Any` (default), `SessionLocal`, `Remote`. Where data or queries may be delivered |

## Query-side types

| Type | Purpose |
|---|---|
| `Selector` | Key expression + `Parameters` (`key/expr?arg1=a;arg2=b`). See [Query](query.md#selectors) |
| `QueryTarget` | `BestMatching` (default), `All`, `AllComplete` |
| `ConsolidationMode` / `QueryConsolidation` | `Auto` (default), `None`, `Monotonic`, `Latest` |
| `ReplyKeyExpr` | :material-flask: `MatchingQuery` (default) or `Any` (accept replies on keys outside the query) |
| `Reply` | `result()` → `Ok(Sample)` or `Err(ReplyError)`. :material-flask: `replier_id()` |
| `ReplyError` | `payload()` + `encoding()` |

## Identity types

| Type | Meaning |
|---|---|
| `ZenohId` | 128-bit runtime ID, written as hex |
| `EntityId` | u32, unique within a session :material-flask: |
| `EntityGlobalId` | `(ZenohId, EntityId)` :material-flask: |
| `WhatAmI` / `WhatAmIMatcher` | `router`/`peer`/`client`, and sets of them |

## Sources

- `zenoh/src/api/sample.rs`, `bytes.rs`, `encoding.rs`, `publisher.rs` (`Priority`), `query.rs`, `selector.rs`
- `commons/zenoh-protocol/src/core/` (`CongestionControl`, `Reliability`, `Priority`, `WhatAmI`)
- `uhlc` (`Timestamp`, `NTP64`)
