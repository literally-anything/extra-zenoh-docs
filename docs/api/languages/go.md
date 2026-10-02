# Go (zenoh-go)

```bash
go get github.com/eclipse-zenoh/zenoh-go
```

zenoh-go wraps **zenoh-c** through cgo. zenoh-c must be built and installed **with
`-DZENOHC_BUILD_WITH_UNSTABLE_API=ON`** (the repository includes it as a git submodule) and be visible to
cgo (include and library paths).

## Packages

| Package | Contents |
|---|---|
| `zenoh` | `Open`, `Scout`, `Config` (`NewConfigDefault`, `NewConfigFromFile`, `NewConfigFromStr`, `NewConfigFromEnv`, `InsertJson5`, `Get`), `Session`, `Publisher`, `Subscriber`, `Queryable`, `Querier`, `Query`, `Reply`, `Liveliness`, `KeyExpr`, `ZBytes`, `Encoding`, `Sample`, `Timestamp`, `SourceInfo`, `CancellationToken`, `Transport`, `Link`, events, `FifoChannel`, `RingChannel`, `Closure` |
| `zenohext` | Serialization, `AdvancedPublisher`, `AdvancedSubscriber`, `SampleMissListener`, heartbeat and miss-detection modes |

## Session methods

`Close`, `IsClosed`, `ZId`, `RoutersZId`, `PeersZId`, `Transports`, `Links`, `Put`, `Delete`, `Get`,
`DeclarePublisher`, `DeclareSubscriber`, `DeclareBackgroundSubscriber`, `DeclareQueryable`,
`DeclareBackgroundQueryable`, `DeclareQuerier`, `Liveliness`, `NewTimestamp`,
`DeclareTransportEventsListener`, `DeclareLinkEventsListener` (and their background variants).

## Conventions

- Options are passed as `*XOptions` structs (`nil` for defaults).
- Handlers: `Closure[T]` (callback), `NewFifoChannel[T](n)`, `NewRingChannel[T](n)`. For channel handlers,
  `subscriber.Handler()` returns a `<-chan Sample`.
- `Undeclare()` / `Drop()` release resources explicitly. Rely on these, not on the garbage collector.

## Not available at 1.10.1

Shared memory.

## Sources

- `zenoh-go@v1.10.1`: `zenoh/*.go`, `zenoh/zenohext/*.go`, `README.md`, `examples/`
