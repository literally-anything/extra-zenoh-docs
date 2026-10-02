# VSOCK

`vsock/` uses Linux **VM sockets** (`AF_VSOCK`) between virtual machines and their hypervisor host, with no
virtual network needed.

| Property | Value |
|---|---|
| Feature | `transport_vsock` (**not** default) |
| Platforms | Linux |
| Reliable / stream | yes / yes |
| Max batch | 65535 |
| io_uring | supported |

## Locator

```text
vsock/<cid>:<port>
```

| Part | Accepted values |
|---|---|
| CID | A number, or `VMADDR_CID_ANY` (or `-1`), `VMADDR_CID_HYPERVISOR`, `VMADDR_CID_LOCAL`, `VMADDR_CID_HOST` (case-insensitive) |
| Port | A number, or `VMADDR_PORT_ANY` (or `-1`) |

```text
vsock/VMADDR_CID_ANY:7447     # listen on all CIDs (inside a VM or on the host)
vsock/VMADDR_CID_HOST:7447    # connect from a guest to the host
vsock/3:7447                  # connect from the host to guest CID 3
```

## Good at / limits

- ✅ Works with no networking in the guest (for example confidential VMs or locked-down sandboxes).
- ⚠️ Linux only, and the hypervisor must expose vsock (`vhost_vsock` on KVM, the Firecracker vsock device, …).

## Sources

- `io/zenoh-links/zenoh-link-vsock/src/`
