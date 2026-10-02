# Authentication

Zenoh authenticates the **remote node** while the transport is being set up. Three mechanisms exist, and
they can be combined:

| Mechanism | Feature | Identity it gives ACL |
|---|---|---|
| TLS / QUIC certificates | `transport_tls`, `transport_quic` | Certificate **common name** (`cert_common_names`) |
| User / password | `auth_usrpwd` (default) | **Username** (`usernames`) |
| RSA public key | `auth_pubkey` (default) | — (see the warning below) |

## TLS / QUIC certificates

See [TLS](../transports/tls.md). In short:

- A listener needs `listen_private_key` + `listen_certificate`.
- With `enable_mtls: true`, the listener requires a client certificate signed by `root_ca_certificate`, and
  the client presents `connect_private_key` + `connect_certificate`.
- The CN of the peer's certificate is recorded per link and can be used in [ACL subjects](access-control.md#subjects).

## User/password

```json5
transport: {
  auth: {
    usrpwd: {
      user: "alice",              // what THIS node presents when it connects
      password: "secret",
      dictionary_file: "/etc/zenoh/users.txt",   // what THIS node accepts
    },
  },
},
```

Dictionary file format: one `user:password` per line.

```text
alice:secret
bob:pw
```

How it works (`ext/auth/usrpwd.rs`):

- The accepting side sends a nonce. The connecting side replies with its user name and an **HMAC of the
  password keyed with that nonce**. The password itself never crosses the network.
- The accepting side looks the user up in its dictionary and checks the HMAC. On failure the transport is
  closed during the handshake.
- `user` and `password` must be set together (config validation fails otherwise). If a node has no
  credentials configured, it doesn't present any.
- Empty users or passwords in the dictionary are rejected: `Invalid user-password dictionary file: empty user.`

!!! note "No encryption"
    User/password authenticates, but it doesn't encrypt. Combine it with `tls/` or `quic/` on untrusted networks.

### Verified behaviour

With a router whose `dictionary_file` contains `alice:secret` and `bob:pw`, and three clients:

| Client | Result |
|---|---|
| `alice` / `secret` | Connected |
| `bob` / `pw` | Connected |
| `eve` / `nope` | Rejected during the handshake: `Received a close message (reason GENERIC) in response to an OpenSyn`. The client then fails to start (`exit_on_failure` is true for clients) |

## Public key

```json5
transport: {
  auth: {
    pubkey: {
      public_key_file: "/etc/zenoh/node.pub",   // PKCS#1 PEM ("BEGIN RSA PUBLIC KEY")
      private_key_file: "/etc/zenoh/node.key",  // PKCS#1 PEM ("BEGIN RSA PRIVATE KEY")
      // or inline: public_key_pem / private_key_pem
      // key_size, known_keys_file: accepted by the schema but NOT used (see below)
    },
  },
},
```

Keys must be **PKCS#1** RSA PEM. With OpenSSL 3:

```bash
openssl genrsa -traditional -out node.key 2048
openssl rsa -in node.key -RSAPublicKey_out -out node.pub
```

!!! danger "Public-key authentication rejects every peer in 1.10.1"
    `AuthPubKey::from_config` loads only this node's own key pair. Where the allowed-keys list should be
    loaded there's a `// @TODO: populate lookup file` comment, so **`known_keys_file` and `key_size` are
    ignored**. The accepting side starts with an **empty** allowed-keys set and rejects every incoming key:

    ```text
    DEBUG zenoh_transport::unicast::establishment::accept:
          PubKey extension - Recv InitSyn. Unauthorized PubKey. at .../ext/auth/pubkey.rs:569
    ```

    The connecting side sees `Received a close message (reason 0) in response to an InitSyn`. We reproduced
    this with two `zenohd` 1.10.1 routers using freshly generated keys. Until it's fixed, use
    **mTLS** or **user/password** for authentication.

## Getting the authenticated identity

- In the API (:material-flask: unstable), `Link::auth_identifier()` returns the link's authenticated
  identity (for example the certificate CN). See [Connectivity events](../discovery/connectivity-events.md).
- In [statistics](../configuration/stats.md), per-transport series carry `remote_cn`.

## Sources

- `commons/zenoh-config/src/lib.rs` (`AuthConf`, `user_conf_validator`)
- `io/zenoh-transport/src/unicast/establishment/ext/auth/` (`usrpwd.rs`, `pubkey.rs`, `mod.rs`)
- `io/zenoh-link-commons/src/tls.rs`, `io/zenoh-links/zenoh-link-tls/src/utils.rs`
