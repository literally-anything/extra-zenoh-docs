# TLS

`tls/` is TCP with TLS (rustls, `ring` crypto provider). Use it for encryption on untrusted networks and
for certificate-based identities that [access control](../security/access-control.md) can match on.

| Property | Value |
|---|---|
| Feature | `transport_tls` (default) |
| Reliable / stream | yes / yes |
| Max batch | Below 65535, aligned to the TCP MSS like [TCP](tcp.md#effective-mtu) |
| TLS versions | TLS 1.2 and 1.3. **TLS 1.3 only** when mTLS is on |
| Private keys | PEM: RSA (PKCS#1), PKCS#8 or EC (SEC1) |
| io_uring | not supported (the TLS layer owns the socket) |

## Locator

```text
tls/router.example.com:7447
```

Use a **hostname** when `verify_name_on_connect` is true (the default): the server certificate is checked
against it.

## Configuration

Settings can go in the global `transport/link/tls` section (applies to every `tls/` **and** `quic/` link)
or on each endpoint after `#`. Endpoint values win.

| Global key (`transport/link/tls/…`) | Endpoint key | Default | Meaning |
|---|---|---|---|
| `root_ca_certificate` | `root_ca_certificate_file` | — | CA certificate file (PEM) used to verify the peer |
| `root_ca_certificate_base64` | `root_ca_certificate_base64` | — | Same, base64 of the PEM |
| — | `root_ca_certificate_raw` | — | Same, PEM inline |
| `listen_private_key` | `listen_private_key_file` / `_raw` / `_base64` | — | Listener key. **Required to listen** |
| `listen_certificate` | `listen_certificate_file` / `_raw` / `_base64` | — | Listener certificate chain. **Required to listen** |
| `connect_private_key` | `connect_private_key_file` / `_raw` / `_base64` | — | Client key (mTLS) |
| `connect_certificate` | `connect_certificate_file` / `_raw` / `_base64` | — | Client certificate (mTLS) |
| `enable_mtls` | `enable_mtls` | `false` | Listener requires and verifies client certificates. The client must also set it to present one |
| `verify_name_on_connect` | `verify_name_on_connect` | `true` | Check the server certificate against the dialled hostname |
| `close_link_on_expiration` | `close_link_on_expiration` | `false` | Close the link when the peer's certificate chain expires |
| `so_sndbuf`, `so_rcvbuf` | `so_sndbuf`, `so_rcvbuf` | OS | TCP buffers |
| — | `tls_handshake_timeout_ms` | `10000` | Timeout for the TLS handshake on accepted connections |

Also accepted per endpoint: `iface`, `bind`, `dscp`.

Setting both a file and a base64 value for the same item is an error
(`Only one between 'root_ca_certificate' and 'root_ca_certificate_base64' can be present!`).

## Trust model

**Connecting side.** The root store is the **Mozilla/WebPKI roots** (`webpki_roots`) **plus** your
`root_ca_certificate`, if set. A server with a publicly trusted certificate is accepted even without a
custom CA. A private-CA deployment must set `root_ca_certificate`.

!!! note "The `DEFAULT_CONFIG.json5` comment is out of date"
    It says WebPKI roots are used "if not specified on router mode". The code (`TlsClientConfig::new`)
    always loads the WebPKI roots, in every mode, and adds the custom CA to them.

**Listening side with mTLS.** Client certificates are checked **only** against `root_ca_certificate`. With
`enable_mtls: true` and no CA, the listener fails: `Missing root certificates while mTLS is enabled.`

**`verify_name_on_connect: false`** skips the hostname check but still checks the chain and expiry. A
warning is logged: `Skipping name verification of TLS server`.

## Certificate expiry

With `close_link_on_expiration: true`, each link starts a timer for the **peer's** certificate chain expiry
and closes the link when it fires. The peer then has to reconnect with a renewed certificate. A listener can
only do this for clients that present a certificate, so mTLS is needed for listener-side expiry.

## Identity for access control

The common name (CN) of the peer certificate becomes the link's auth identifier. ACL subjects match on it
with `cert_common_names`. See [Access control](../security/access-control.md).

## Example

=== "Router (server)"

    ```json5
    {
      mode: "router",
      listen: { endpoints: ["tls/0.0.0.0:7447"] },
      transport: { link: { tls: {
        root_ca_certificate: "/etc/zenoh/ca.pem",
        listen_private_key: "/etc/zenoh/router-key.pem",
        listen_certificate: "/etc/zenoh/router-cert.pem",
        enable_mtls: true,
      } } },
    }
    ```

=== "Client"

    ```json5
    {
      mode: "client",
      connect: { endpoints: ["tls/router.example.com:7447"] },
      transport: { link: { tls: {
        root_ca_certificate: "/etc/zenoh/ca.pem",
        enable_mtls: true,
        connect_private_key: "/etc/zenoh/client-key.pem",
        connect_certificate: "/etc/zenoh/client-cert.pem",
      } } },
    }
    ```

## Sources

- `io/zenoh-links/zenoh-link-tls/src/` (`utils.rs`: `TlsConfigurator`, `TlsServerConfig`, `TlsClientConfig`)
- `io/zenoh-link-commons/src/tls.rs` (config keys, defaults, expiration manager)
