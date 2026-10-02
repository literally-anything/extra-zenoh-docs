# Kotlin (zenoh-kotlin)

```kotlin
// JVM
implementation("org.eclipse.zenoh:zenoh-kotlin:<version>")
// Android
implementation("org.eclipse.zenoh:zenoh-kotlin-android:<version>")
```

A Kotlin Multiplatform library (JVM and Android targets) over the Rust core through JNI. Most calls return
`kotlin.Result<T>`.

## API surface (`io.zenoh.*`)

| Area | Types / functions |
|---|---|
| Entry | `Zenoh.open(config)`, `Zenoh.scout(...)`, `Config.default()`, `fromFile`, `fromJson`, `fromJson5`, `fromYaml`, `fromEnv`, `insertJson5`, `getJson` |
| Session | `declarePublisher`, `declareSubscriber`, `declareQueryable`, `declareQuerier`, `declareKeyExpr`, `get`, `put`, `delete`, `liveliness()`, `info()` (`zid`, `peersZid`, `routersZid`), `isClosed`, `close` |
| Pub/sub | `Publisher` (`put`, `delete`, QoS getters), `Subscriber` |
| Query | `Query` (`reply`, `replyErr`, `replyDel`), `Queryable`, `Querier`, `Reply`, `Selector`, `Parameters`, `QueryTarget`, `ConsolidationMode`, `ReplyKeyExpr` |
| Liveliness | `Liveliness.declareToken`, `declareSubscriber`, `get` |
| Data | `ZBytes`, `Encoding`, `Sample`, `Timestamp`, `QoS`, `Priority`, `CongestionControl`, `Reliability` |
| Handlers | `Callback`, `ChannelHandler` (Kotlin coroutines `Channel`), custom `Handler<T, R>` |
| Serialization | `zSerialize<T>(t)` / `zDeserialize<T>(bytes)` (JVM/Android) |
| `@Unstable` | `declareAdvancedPublisher`, `declareAdvancedSubscriber`, `CacheConfig`, `HistoryConfig`, `RecoveryConfig`, `MissDetectionConfig`, sample-miss listeners, matching listeners on `AdvancedPublisher` |

## Not available at 1.10.1

- Matching listeners on plain `Publisher` / `Querier` (only on `AdvancedPublisher`)
- Transport/link info and events
- Shared memory
- Cancellation tokens
- `newTimestamp` on the session

## Sources

- `zenoh-kotlin@1.10.1`: `zenoh-kotlin/src/commonMain/kotlin/io/zenoh/`, `jvmAndAndroidMain/…/ext/`, `README.md`
