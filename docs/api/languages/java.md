# Java (zenoh-java)

```kotlin
implementation("org.eclipse.zenoh:zenoh-java:<version>")          // JVM
implementation("org.eclipse.zenoh:zenoh-java-android:<version>")  // Android
```

zenoh-java is written in Kotlin but designed to be called from Java: exceptions instead of `Result`,
option objects instead of named parameters, and `BlockingQueue`-based handlers. It uses JNI over the Rust core.

## API surface (`io.zenoh.*`)

| Area | Types / functions |
|---|---|
| Entry | `Zenoh.open(config)`, `Zenoh.scout(...)`, `Config.loadDefault()`, `fromFile`, `fromJson`, `fromJson5`, `fromYaml`, `fromEnv`, `insertJson5`, `getJson` |
| Session | `declarePublisher(ke, PublisherOptions)`, `declareSubscriber(ke[, callback])`, `declareQueryable`, `declareQuerier`, `declareKeyExpr`, `get(selector, GetOptions)`, `put(ke, payload, PutOptions)`, `delete(ke, DeleteOptions)`, `liveliness()`, `info()` (`zid`, `peersZid`, `routersZid`), `close` |
| Query | `Query.reply(ke, payload, ReplyOptions)`, `replyErr`, `replyDel`, `Queryable`, `Querier`, `Reply`, `Selector`, `Parameters` |
| Liveliness | `declareToken`, `declareSubscriber`, `get` |
| Data | `ZBytes`, `Encoding`, `Sample`, `Timestamp`, `Priority`, `CongestionControl`, `Reliability` |
| Handlers | `Callback<T>`, `BlockingQueueHandler` (the default: a `BlockingQueue<Optional<T>>`), custom `Handler<T, R>` |
| Serialization | `ZSerializer<T>` / `ZDeserializer<T>` with a type token (`io.zenoh.ext`) |

## Not available at 1.10.1

- Matching status and listeners
- Advanced pub/sub
- Transport/link info and events
- Shared memory, cancellation tokens, session `newTimestamp`

## Sources

- `zenoh-java@1.10.1`: `zenoh-java/src/commonMain/kotlin/io/zenoh/`, `examples/`, `README.md`
