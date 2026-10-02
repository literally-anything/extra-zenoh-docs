# Example programs (`z_*`)

The Zenoh repository ships a set of small command-line programs, the `z_*` examples. They're the quickest
way to test a deployment, a config or a link: `z_sub` on one machine, `z_pub` on another, and you can see
whether data flows. The bindings ship the same programs, with the same names and mostly the same options.

```bash
# from a zenoh checkout
cargo run --release --example z_sub
cargo run --release --example z_pub -- -e tcp/192.168.1.10:7447
# SHM examples need extra features
cargo run --release --example z_pub_shm --features shared-memory,unstable
# zenoh-ext examples
cargo run --release -p zenoh-ext-examples --example z_advanced_sub
```

## Common options

Every example except `z_scout`, `z_formats`, `z_bytes`, `z_bytes_shm`, `z_alloc_shm`,
`z_posix_shm_provider` and `z_member` accepts these (`examples/src/lib.rs`, `CommonArgs`):

| Option | Effect |
|---|---|
| `-c, --config <FILE>` | Load a config file |
| `--cfg <KEY:VALUE>` | Set any config key; VALUE is JSON5. Repeatable. Example: `--cfg='transport/unicast/max_links:2'` |
| `-m, --mode <peer\|client\|router>` | Session mode (default **peer**) |
| `-e, --connect <ENDPOINT>` | Endpoint to connect to. Repeatable |
| `-l, --listen <ENDPOINT>` | Endpoint to listen on. Repeatable |
| `--no-multicast-scouting` | Sets `scouting/multicast/enabled: false` |
| `--enable-shm` | Sets `transport/shared_memory/enabled: true`. Exits with an error if built without `shared-memory` |

`--cfg` is applied last, so it overrides everything else. It splits on the **first** `:` only, so
`--cfg='connect/endpoints:["tcp/10.0.0.1:7447"]'` works.

## Programs

| Program | What it does | Own options (defaults) |
|---|---|---|
| `z_scout` | Scouts for peers and routers for 1 s and prints each Hello | — |
| `z_info` | Prints own ZID and the ZIDs of connected routers and peers (and, when built with `--features unstable`, transports and links) | — |
| `z_put` | One `put` | `-k demo/example/zenoh-rs-put`, `-p "Put from Rust!"` |
| `z_put_float` | One `put` of an `f64` serialized with zenoh-ext | `-k …`, `-p 3.14159…` |
| `z_delete` | One `delete` | `-k demo/example/zenoh-rs-put` |
| `z_pub` | Publishes once per second | `-k demo/example/zenoh-rs-pub`, `-p "Pub from Rust!"`, `-a <attachment>`, `--add-matching-listener` |
| `z_sub` | Subscribes and prints samples | `-k demo/example/**` |
| `z_pull` | Ring-channel subscriber read every N seconds (shows "latest value" semantics) | `-k demo/example/**`, `-s 3` (ring size), `-i 5.0` (interval, s) |
| `z_queryable` | Answers queries | `-k demo/example/zenoh-rs-queryable`, `-p …`, `--complete` |
| `z_get` | One query, prints replies | `-s demo/example/**`, `-p <payload>`, `-t BEST_MATCHING\|ALL\|ALL_COMPLETE`, `-o 10000` (timeout, ms) |
| `z_querier` | Declares a querier and queries once per second | as `z_get`, plus `--add-matching-listener` |
| `z_storage` | A tiny in-memory storage: subscriber and queryable on the same key | `-k demo/example/**`, `--complete` |
| `z_forward` | Re-publishes everything from one key on another (uses `SubscriberForward`) | `-k demo/example/**`, `-f demo/forward` |
| `z_liveliness` | Declares a liveliness token | `-k group1/zenoh-rs` |
| `z_sub_liveliness` | Liveliness subscriber | `-k group1/**`, `--history` |
| `z_get_liveliness` | Liveliness query | `-k group1/**`, `-o 10000` |
| `z_formats` | Shows `kedefine!`/`keformat!` key-expression formats | — |
| `z_bytes` | Shows `ZBytes` and zenoh-ext serialization (no network) | — |
| `z_pub_thr` / `z_sub_thr` | Throughput test on `test/thr` | pub: `<PAYLOAD_SIZE>`, `--express`, `-p <priority>`, `-t` (print), `-n 100000`; sub: `-s 10` (rounds), `-n 100000` (messages per round) |
| `z_ping` / `z_pong` | Round-trip latency on `test/ping` / `test/pong` | ping: `<PAYLOAD_SIZE>`, `-w 1` (warm-up, s), `-n 100` (samples), `--no-express`; pong: `--no-express` |

Shared-memory variants (need `--features shared-memory,unstable`): `z_pub_shm`, `z_sub_shm`,
`z_queryable_shm`, `z_get_shm`, `z_pub_shm_thr` (`-s 32` MiB pool), `z_ping_shm`, `z_alloc_shm`
(allocation policies), `z_bytes_shm` (SHM buffer API), and `z_posix_shm_provider` (making a provider).

zenoh-ext examples (`-p zenoh-ext-examples`):

| Program | Options (defaults) |
|---|---|
| `z_advanced_pub` | `-k demo/example/zenoh-rs-pub`, `-v "Pub from Rust!"`, `-i 1` (history depth) |
| `z_advanced_sub` | `-k demo/example/**` |
| `z_member` | Joins group `zgroup` with a 3 s lease and prints events, view and leader ([Group](../api/zenoh-ext-legacy.md#group-membership-zenoh_extgroup)) |
| `z_view_size` | `-g zgroup`, `-s 3` (expected size), `-t 15` (timeout, s), `-i <member id>` |

## Typical checks

```bash
# Does traffic go through my router?
z_sub -m client -e tcp/router:7447 &
z_pub -m client -e tcp/router:7447

# Is a link type usable between two hosts? (no scouting, so only the explicit endpoint is used)
z_sub --no-multicast-scouting -l quic/0.0.0.0:7447 --cfg 'transport/link/tls:{…}'
z_pub --no-multicast-scouting -e quic/host-a:7447 --cfg 'transport/link/tls:{…}'

# Throughput and latency
z_sub_thr &   z_pub_thr 8
z_pong &      z_ping 64
```

## Sources

- `examples/Cargo.toml` (`[[example]]`, `required-features`), `examples/src/lib.rs` (`CommonArgs`)
- `examples/examples/*.rs`, `zenoh-ext/examples/examples/*.rs`
- Checked with `--help` on the 1.10.1 builds
