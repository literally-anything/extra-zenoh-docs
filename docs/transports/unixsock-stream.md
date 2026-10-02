# Unix socket stream

`unixsock-stream/` uses Unix domain stream sockets for processes on the same host.

| Property | Value |
|---|---|
| Feature | `transport_unixsock-stream` (default) |
| Platforms | Unix (Linux, macOS, BSD) |
| Reliable / stream | yes / yes |
| Max batch | 65535 |
| io_uring | supported |

## Locator

The address is a filesystem path. Absolute paths start with `/`, so the locator has a double slash:

```text
unixsock-stream//tmp/zenoh.sock
unixsock-stream//run/zenoh/router.sock
```

## Lock file

Unix sockets have no `SO_REUSEADDR`, so the listener creates a lock file next to the socket
(`<path>.lock`, mode `0600`) and takes an exclusive lock on it:

- Lock taken → nobody is using the socket, so a stale socket file is removed and the listener binds.
- Lock held by another process → listening fails.

The kernel releases the lock if the owner exits or crashes, so stale sockets from a crashed process are
cleaned up on the next start.

## Good at / limits

- ✅ Lower overhead than loopback TCP. File permissions control who can connect.
- ⚠️ Same host only. Payloads are still copied. For zero-copy large payloads, combine it with [shared memory](../shm/index.md).

## Sources

- `io/zenoh-links/zenoh-link-unixsock_stream/src/`
