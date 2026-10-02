# Serial

`serial/` runs Zenoh over a UART or USB serial port (the `z-serial` crate). It's the usual way to connect
microcontrollers running [zenoh-pico](../pico/transports.md#serial) to a host or router.

| Property | Value |
|---|---|
| Feature | `transport_serial` (**not** default) |
| Reliable | **no** (CRC-checked frames, no retransmission) |
| Byte stream | no (framed) |
| Max batch | **1500 bytes** (`z_serial::MAX_MTU`) |
| io_uring | not supported |

## Locator

```text
serial//dev/ttyUSB0#baudrate=115200
serial/COM3#baudrate=921600
```

## Config (`#`) options

| Key | Default | Meaning |
|---|---|---|
| `baudrate` | `9600` | Line speed |
| `exclusive` | `true` | Open the port exclusively |
| `tout` | `50000` | Connect timeout in **microseconds** |
| `release_on_close` | `true` | Release the port when the link closes |

Unparseable values silently fall back to the default.

## Wire format

Each Zenoh batch (≤ 1500 bytes) becomes one frame:

```
| COBS overhead | len (2) | kind (1) | data (≤1500) | CRC32 (4) | 0x00 sentinel |
```

- **COBS** encoding removes zeros from the frame so `0x00` can mark frame boundaries.
- **CRC32** (polynomial `0x04C11DB7`) catches corrupt frames, which are dropped.
- A small handshake (init / ack / reset flags) runs on connect, retried every 250 ms.

## Good at / limits

- ✅ Connects devices with no IP stack at all.
- ⚠️ Low bandwidth: at 115200 baud about 11 KB/s before overhead. Use [downsampling](../configuration/downsampling.md)
  and [low-pass filters](../configuration/low-pass-filter.md) to protect the link.
- ⚠️ Messages over 1500 bytes are fragmented, and losing one fragment loses the message.
- ⚠️ Not in stock `zenohd`. Rebuild with `--features zenoh/transport_serial`.

## Sources

- `io/zenoh-links/zenoh-link-serial/src/`
- `z-serial` 0.3.1 (`MAX_MTU = 1500`, COBS, CRC32)
