# Unix pipe

`unixpipe/` uses **named pipes (FIFOs)** for communication on the same host.

| Property | Value |
|---|---|
| Feature | `transport_unixpipe` (**not** default) |
| Platforms | Unix |
| Reliable / stream | yes / yes |
| Max batch | 65535 |
| io_uring | supported |

## Locator

```text
unixpipe//tmp/zenoh-pipe
```

## How it connects

1. The listener creates a **request channel**, a named pipe at the given path.
2. A connecting process picks a random suffix and creates a dedicated **uplink** and **downlink** pipe pair.
   It retries up to 100 times to find an unused suffix.
3. It sends an invitation naming the pair over the request channel.
4. The listener opens the pair and confirms on it. The link is now up.

Pipes are locked so only one party uses each one.

## Configuration

| Where | Key | Default | Meaning |
|---|---|---|---|
| Endpoint `#` | `file_mask` | `0o777` | Permissions mask for created pipe files |
| Global | `transport/link/unixpipe/file_access_mask` | `0o777` | Same, for every unixpipe link |

`transport/link/unixpipe/file_access_mask` isn't in the upstream `DEFAULT_CONFIG.json5`.

!!! warning "Permissive default"
    The default mask `0o777` lets any local user open the pipes. Set a tighter mask (for example `0o600`) on
    multi-user hosts.

## Sources

- `io/zenoh-links/zenoh-link-unixpipe/src/unix/` (`mod.rs`, `unicast.rs`)
