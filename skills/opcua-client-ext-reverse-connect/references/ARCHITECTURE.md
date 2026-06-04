# Architecture

How the pieces fit, and why the design is shaped this way.

## The Reverse Connect flow (Part 6 §7.1.2.3)

In normal OPC UA the client opens the TCP connection. In **Reverse Connect** the roles invert at the transport layer: the **server** opens the TCP connection to the client and immediately sends a `ReverseHello` (`RHE`) frame announcing its `ServerUri` and the `EndpointUrl` to use. After that single inversion, the protocol continues exactly as usual — HEL/ACK, OpenSecureChannel, CreateSession, ActivateSession — all on the **same socket** the server dialed.

This solves the topology where the server (an edge gateway, a PLC behind NAT/firewall) cannot accept inbound connections but *can* make outbound ones to a known, reachable client.

```
Server (behind NAT)                         Client (public/reachable)
      │                                            │
      │  outbound TCP connect ───────────────────► │  ReverseConnectListener
      │                                            │    (bound on host:port)
      │  RHE { ServerUri, EndpointUrl } ─────────► │  parse + validate
      │                                            │
      │ ◄──────── HEL / ACK ──────────────────────►│  TcpTransport (same socket)
      │ ◄──────── OPN (secure channel) ───────────►│
      │ ◄──────── CreateSession / Activate ───────►│  Client (core)
      │ ◄──────── Read / Write / Browse / Sub ────►│
```

## Components

| Component | Responsibility | I/O? |
| --- | --- | --- |
| `ReverseConnectListener` | Owns the TCP server socket. Accepts one inbound connection per `accept()`, reads the framed RHE off the wire, runs the validator, emits events, returns a `ReverseConnectSession`. | Yes (sockets) |
| `ReverseHelloParser` | Pure function: bytes → `ReverseHelloMessage`. Enforces all structural rules. | No |
| `ReverseHelloMessage` | Immutable decoded payload (`serverUri`, `endpointUrl`). | No |
| `ReverseHelloValidator` | Trust decision: whitelist match + scheme check. The security gate. | No |
| `ReverseConnectSession` | Immutable carrier of the validated identity + the live socket. | No |
| `ReverseConnectClientFactory` | Wraps the socket in a core `TcpTransport` and runs `ClientBuilder::connect()`. | No (socket already open) |

The split keeps the **only impure component** (the listener) thin, and makes the parser/validator trivially unit-testable without sockets.

## The RHE wire frame

```
+--------+--------+----------------+-------------+--------------+
| "RHE"  |  "F"   |   MessageSize  |  ServerUri  |  EndpointUrl |
| 3 byte | 1 byte |  4 byte UInt32 |  OPC UA Str |  OPC UA Str  |
+--------+--------+----------------+-------------+--------------+
        8-byte standard header             length-prefixed strings
```

- `MessageType` must be exactly `RHE`; `ChunkType` must be `F` (final — RHE is never chunked).
- `MessageSize` is the **total** frame length (header included), little-endian UInt32. The parser requires it to equal the received length exactly — no trailing bytes, no short frames.
- Each OPC UA String is an Int32 LE length prefix + UTF-8 bytes; length `-1` is the OPC UA "null string" and is normalised to `''`. (The validator then rejects empty `ServerUri`/`EndpointUrl`, so a null URL never reaches the transport.)
- `ReverseHelloParser` reuses the core's `Encoding\BinaryDecoder::readString()` for the string fields, so OPC UA string semantics stay consistent with the rest of `opcua-client`.

The listener reads the frame in two steps (`readFrame()`): first the 8-byte header to learn `MessageSize`, then exactly `MessageSize - 8` more bytes, bounded by `maxFrameSize`. This avoids over-reading into a following frame and caps memory per the `maxFrameSize` ctor argument (default 65535).

## The socket handoff — why no second connection

This is the crux of the integration with the core. After `accept()`, the socket is **already TCP-connected** (the server dialed it) and **no UA frame has been written yet** — exactly the state a normal client `connect()` would have produced right before sending HEL.

`ReverseConnectClientFactory::buildClient()`:

1. `TcpTransport::fromConnectedSocket($session->socket, $readTimeout)` — wraps the existing resource instead of opening a new one.
2. `ClientBuilder::create()->setTransport($transport)` — injects it via the `ClientTransportInterface` seam (added in core v4.4.0).
3. runs the optional `$configure(ClientBuilder)` callback (security, identity, logger, dispatcher, extra modules).
4. `->connect($session->endpointUrl)` — the core's `ManagesConnectionTrait::performConnect()` checks `transport->isConnected()`, finds it `true`, and **skips the outbound `connect($host, $port)`**, proceeding straight to the UA-TCP handshake on the inherited socket.

If you instead called `ClientBuilder::connect($endpointUrl)` yourself, the core *would* dial outbound to `endpointUrl` — which is wrong: the server may be unreachable from the client side (that's the whole reason Reverse Connect exists), and you'd be throwing away the socket the server already gave you.

## Dependency boundary with the core

- Hard requirement: `php-opcua/opcua-client ^4.4`. The `fromConnectedSocket()` factory and the already-connected skip are both v4.4.0 additions.
- Reused core types: `Encoding\BinaryDecoder`, `Exception\EncodingException`, `Transport\TcpTransport`, `Client`, `ClientBuilder`.
- This package adds **no** OPC UA service logic. It ends where the core begins: a connected `Client`.

See [`LISTENER.md`](LISTENER.md) for the operational API and [`SECURITY.md`](SECURITY.md) for the trust model.
