# Security

Zenoh security has three layers:

| Layer | What it gives you | Mechanisms |
|---|---|---|
| **Transport encryption** | Confidentiality and integrity on the wire | [TLS](../transports/tls.md), [QUIC](../transports/quic.md) |
| **Authentication** | Who is connecting | TLS/QUIC certificates (mTLS), [user/password](authentication.md#userpassword), [RSA public key](authentication.md#public-key) |
| **Authorization** | What they may do | [Access control (ACL)](access-control.md) |

Interceptors such as [downsampling](../configuration/downsampling.md) and [low-pass filters](../configuration/low-pass-filter.md)
also protect resources.

## Recommended baseline for untrusted networks

1. Routers listen only on `tls/` or `quic/` (use `transport/link/protocols` to disable everything else).
2. `enable_mtls: true` with a private CA, so every client has a certificate whose CN identifies it.
3. ACL `enabled: true`, `default_permission: "deny"`, with explicit allow rules per CN or username.
4. Lock down the [admin space](../admin-space/index.md): `--adminspace-permissions r` (or `none`), and ACL
   rules on `@/**`.
5. Turn off multicast scouting on exposed interfaces (`scouting/multicast/enabled: false`) and list endpoints explicitly.

## Things to know

- **ZIDs aren't authenticated.** Anyone can set any `id`. Don't use ZID subjects in ACL for production
  ([the config docs](access-control.md#subjects) say the same).
- **Public-key authentication can't accept peers in 1.10.1.** See the [warning](authentication.md#public-key).
- ACL and the other interceptors apply to **unicast transports only**, not to multicast groups and not to
  traffic between entities in the same session.
- `private` config keys are hidden from logs and the admin space, and TLS `*_base64` secrets are never serialized.

## Sources

- See the [Authentication](authentication.md) and [Access control](access-control.md) pages.
