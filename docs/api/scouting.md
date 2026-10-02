# Scouting API

`scout()` runs [multicast scouting](../discovery/scouting.md) **without opening a session**. It's useful
for discovery tools and for choosing an endpoint before connecting.

```rust
use zenoh::config::WhatAmI;

let receiver = zenoh::scout(WhatAmI::Router | WhatAmI::Peer, zenoh::Config::default()).await?;
while let Ok(hello) = receiver.recv_async().await {
    println!("{} {} {:?}", hello.zid(), hello.whatami(), hello.locators());
}
```

- The first argument is a `WhatAmIMatcher`: which modes should answer.
- The config supplies `scouting/multicast/address`, `interface` and `ttl`.
- Each answer is a `Hello` with `zid()`, `whatami()` and `locators()`.
- Scouts are resent with backoff (1 s doubling to 8 s) until you drop the `Scout` handle or stop it.

| Language | Call |
|---|---|
| C / Pico | `z_scout(config, closure, options)` (`z_hello_zid`, `z_hello_whatami`, `z_hello_locators`) |
| C++ | `zenoh::scout(config, callback, on_drop, options)` |
| Python | `zenoh.scout(handler, what="peer|router", config=…)` |
| Kotlin / Java | `Zenoh.scout(...)` |
| Go | `zenoh.Scout(config, handler, options)` |
| TypeScript | ❌ (remote-only binding) |

## Sources

- `zenoh/src/api/scouting.rs`, `commons/zenoh-config/src/wrappers.rs` (`Hello`)
- `zenoh/src/net/runtime/orchestrator.rs` (`scout`)
