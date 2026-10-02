# Python (zenoh-python)

```bash
pip install eclipse-zenoh          # imports as `zenoh`
```

Built with PyO3 on the Rust core. The published wheels include the APIs marked unstable (they carry an
`_unstable` marker in the docs) and the SHM module.

## Modules

| Module | Contents |
|---|---|
| `zenoh` | `open`, `scout`, `Session`, `Config`, `KeyExpr`, `Publisher`, `Subscriber`, `Querier`, `Queryable`, `Query`, `Reply`, `ReplyError`, `Sample`, `ZBytes`, `Encoding`, `Priority`, `CongestionControl`, `Reliability`, `Locality`, `QueryTarget`, `ConsolidationMode`, `Selector`, `Parameters`, `Timestamp`, `ZenohId`, `Liveliness`, `LivelinessToken`, `SessionInfo`, `Transport`, `Link`, events, `CancellationToken`, `TimestampInstrumentationBuilder`, `TimestampStack`, `InterceptionPoint` ([timestamp stack](../timestamp-stack.md); `open(config, timestamp_callback=…)`) |
| `zenoh.handlers` | `DefaultHandler`, `FifoChannel`, `RingChannel`, `Callback`, `Handler` |
| `zenoh.ext` | `z_serialize`, `z_deserialize`, width types (`Int8`…`UInt128`, `Float32`, `Float64`), `declare_advanced_publisher`, `declare_advanced_subscriber`, `CacheConfig`, `HistoryConfig`, `RecoveryConfig`, `MissDetectionConfig`, `RepliesConfig`, `Miss`, `SampleMissListener` |
| `zenoh.shm` | `ShmProvider`, `MemoryLayout`, `AllocAlignment`, policies (`JustAlloc`, `BlockOn`, `GarbageCollect`, `Defragment`, `Deallocate`), `ZShm`, `ZShmMut` |

## Example

```python
import zenoh

with zenoh.open(zenoh.Config.from_file("peer.json5")) as session:
    pub = session.declare_publisher("demo/temp", priority=zenoh.Priority.DATA_HIGH)
    sub = session.declare_subscriber("demo/**", lambda s: print(s.key_expr, s.payload.to_string()))
    pub.put("21.5")
    for reply in session.get("demo/**", timeout=2.0):
        print(reply.ok.payload.to_string() if reply.ok else reply.err.payload.to_string())
```

## Conventions

- Options are **keyword arguments**, not builders.
- Entities and sessions are context managers (`with …:`), and leaving the block undeclares or closes them.
- Handlers: pass a callable (runs on a Zenoh thread with the GIL taken), a `FifoChannel(n)`/`RingChannel(n)`,
  or nothing (the default FIFO). Channel-backed objects are iterable and have `recv()`/`try_recv()`.
- Logging: `zenoh.init_log_from_env_or("error")` or `zenoh.try_init_log_from_env()`.

## Sources

- `zenoh-python@1.10.1`: `zenoh/__init__.pyi`, `zenoh/ext.pyi`, `zenoh/shm.pyi`, `zenoh/handlers.pyi`, `pyproject.toml`
