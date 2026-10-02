# Serialization

Zenoh carries plain bytes. For typed data **between languages**, `zenoh-ext` defines a small binary
format that every binding implements, so a Python tuple can be read as a Rust tuple or a C struct.
Payloads in this format should use the encoding `zenoh/serialized`.

## Wire format

From `zenoh-ext/src/serialization.rs` (spec: the Zenoh roadmap RFC "Serialization"):

| Type | Encoding |
|---|---|
| Integers (`u8`…`u64`, `i8`…`i64`, `u128`/`i128` where supported) | Fixed width, **little-endian** |
| `f32`, `f64` | IEEE 754, little-endian |
| `bool` | 1 byte (`0`/`1`) |
| Length prefixes | **LEB128 varint** (`VarInt<usize>`) |
| `String` / `&str` | varint length + UTF-8 bytes |
| Bytes / `Vec<u8>` / slices / sequences | varint length + elements |
| Fixed-size arrays `[T; N]` | varint length (must equal N) + elements |
| Tuples / structs | Fields concatenated in order, no tags |
| Maps | varint length + key, value pairs |

There are no type tags. Both sides must agree on the schema.

## Rust

```rust
use zenoh_ext::{z_serialize, z_deserialize, ZSerializer, ZDeserializer};

let bytes = z_serialize(&(42i32, vec![1u8, 2, 3], "hi".to_string()));
let (a, b, c): (i32, Vec<u8>, String) = z_deserialize(&bytes)?;

// Streaming
let mut s = ZSerializer::new();
s.serialize(1.5f64);
s.serialize("abc");
let bytes = s.finish();
let mut d = ZDeserializer::new(&bytes);
let x: f64 = d.deserialize()?;
let y: String = d.deserialize()?;
assert!(d.done());
```

Implement the `Serialize`/`Deserialize` traits for your own types.

## Other languages

| Language | API |
|---|---|
| C | `ze_serialize_*` / `ze_deserialize_*` one-shots, and `ze_serializer_*` / `ze_deserializer_*` streaming (int8…uint64, float, double, string, slice, sequence length) |
| C++ | `zenoh::ext::serialize` / `deserialize`, `Serializer` / `Deserializer` (STL containers, tuples, custom types) |
| Python | `zenoh.ext.z_serialize(obj)` / `z_deserialize(type, zbytes)`. Explicit width wrappers `Int8`…`UInt128`, `Float32`, `Float64` |
| Kotlin | `zSerialize(t)` / `zDeserialize<T>(bytes)` (reified generics, JVM/Android) |
| Java | `ZSerializer<T>` / `ZDeserializer<T>` with a type token |
| TypeScript | `zserialize` / `zdeserialize`, `ZBytesSerializer` / `ZBytesDeserializer`, `ZS`/`ZD` helpers, `NumberFormat`/`BigIntFormat` to pick widths |
| Pico | `ze_serialize_*` / `ze_deserialize_*` (same as C) |
| Go | `zenohext` serialization helpers |

!!! tip "Pick widths explicitly in dynamic languages"
    Python, TypeScript and Java have no native fixed-width integer types. Use the width wrappers
    (`Int32`, `NumberFormat.Int32`, …) when talking to a statically typed peer.

## Sources

- `zenoh-ext/src/serialization.rs`, `zenoh-ext/src/lib.rs`
- `zenoh-c/include/zenoh_commons.h` (`ze_*serializ*`), `zenoh-python/zenoh/ext.pyi`,
  `zenoh-ts/zenoh-ts/src/ext/`, `zenoh-kotlin/.../ext/ZSerialize.kt`
