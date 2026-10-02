# SHM API

The Rust SHM API lives in `zenoh::shm` and needs the `shared-memory` **and** `unstable` features.
The C, C++ and Python bindings mirror these concepts (see [Languages](../api/index.md)).

## Concepts

| Type | Role |
|---|---|
| `ShmProviderBackend` | Owns SHM segments and allocates chunks inside them |
| `PosixShmProviderBackend` | Default backend: **talc** allocator on POSIX SHM |
| `PosixShmProviderBackendBuddy` | Buddy allocator: fastest, more fragmentation, less memory-efficient |
| `PosixShmProviderBackendBinaryHeap` | Legacy largest-fit allocator. Keeps **no metadata in SHM**, so receivers can't corrupt it |
| `ShmProvider` | Front end for allocations, with policies |
| `MemoryLayout` / `TypedLayout<T>` | Size + alignment (`AllocAlignment`) of allocations |
| `ZShmMut` / `zshmmut` | Mutable SHM buffer (you hold the only reference) |
| `ZShm` / `zshm` | Immutable SHM buffer (shared) |
| `Typed<T, Buf>` | Typed view of a buffer, for `#[repr(C)]` types that implement `unsafe ResideInShm` |
| `ShmClient` / `ShmClientStorage` | Receiving side: knows how to open segments of each protocol ID |

## Creating a provider

```rust
use zenoh::{shm::{ShmProviderBuilder, PosixShmProviderBackend, AllocAlignment}, Wait};

// Simple: default (talc) backend with 1 MiB of SHM
let provider = ShmProviderBuilder::default_backend(1024 * 1024).wait()?;

// Explicit backend and alignment
let backend = PosixShmProviderBackend::builder((65536, AllocAlignment::ALIGN_8_BYTES)).wait()?;
let provider = ShmProviderBuilder::backend(backend).wait();
```

## Allocating

```rust
// Direct allocation (layout computed each time)
let mut buf = provider.alloc(512).wait()?;
let buf = provider.alloc((512, AllocAlignment::ALIGN_2_BYTES)).wait()?;

// Reusable layout for many allocations of the same shape (faster)
let layout = provider.alloc_layout(512)?;
let buf = layout.alloc().wait()?;

// Typed allocation
#[repr(C)] struct Frame { len: AtomicUsize, data: [u8; 1024] }
unsafe impl zenoh::shm::ResideInShm for Frame {}
let typed = provider.alloc(TypedLayout::<Frame>::new()).wait()?;   // Typed<MaybeUninit<Frame>, ZShmMut>
```

### Allocation policies

Policies are generic types, so the compiler picks the behaviour when memory runs out:

| Policy | Behaviour when there's no free space |
|---|---|
| `JustAlloc` (default) | Fail immediately |
| `GarbageCollect<…>` | Reclaim chunks nobody references, then retry |
| `Defragment<…>` | Defragment, then retry |
| `Deallocate<N, …>` (unsafe) | Forcibly deallocate up to N chunks (`DeallocateYoungest`, `DeallocateEldest`, `DeallocateOptimal`), then retry |
| `BlockOn<…>` | Wait (sync with `.wait()` or async with `.await`) until memory frees up |

They compose:

```rust
let buf = provider.alloc(512)
    .with_policy::<BlockOn<Defragment<GarbageCollect>>>()
    .await?;
```

## Publishing

An SHM buffer converts into `ZBytes` like any payload:

```rust
let mut sbuf = provider.alloc(len).with_policy::<BlockOn<GarbageCollect>>().await?;
sbuf[..len].copy_from_slice(data);
publisher.put(sbuf).await?;
```

`put`, `reply`, query payloads and attachments all accept SHM buffers.

## Receiving

Receiving code doesn't need to change. A `ZBytes` backed by SHM reads like any other. To check:

```rust
match sample.payload_mut().as_shm_mut() {
    Some(shm) => match <&mut zshmmut>::try_from(shm) {
        Ok(_m) => "SHM (mutable)",     // we hold the only reference
        Err(_) => "SHM (immutable)",
    },
    None => "RAW",                       // came over the network or was copied
}
```

If zenoh is built with `shared-memory` but without `unstable`, SHM buffers are received and read normally,
but you can't tell them apart from regular ones.

## Provider state

`zenoh::shm::ShmProviderState` exposes the runtime's internal provider, the one used for implicit SHM.
It's `lazy` or `init` depending on `transport/shared_memory/mode`.

## Examples in the repo

`examples/examples/`: `z_alloc_shm.rs`, `z_pub_shm.rs`, `z_sub_shm.rs`, `z_pub_shm_thr.rs`, `z_ping_shm.rs`,
`z_get_shm.rs`, `z_queryable_shm.rs`, `z_bytes_shm.rs`, `z_posix_shm_provider.rs`.

## Sources

- `zenoh/src/lib.rs` (`pub mod shm`, gated by `unstable` + `shared-memory`)
- `commons/zenoh-shm/src/api/` (`provider/`, `buffer/`, `protocol_implementations/posix/`)
