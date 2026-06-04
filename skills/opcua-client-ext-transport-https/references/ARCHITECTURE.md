# Architecture

How `HttpsTransport` makes the unchanged core `Client::connect()` pipeline run over HTTPS.

## The §7.4 model vs UA-TCP

`opc.tcp://` is a stateful, framed, chunked protocol with its own secure channel (OpenSecureChannel). The OPC UA **HTTPS mappings (Part 6 §7.4)** replace all of that with HTTP:

- **One HTTPS POST per UA service message.** No persistent UA framing on the wire, no MSGF chunking, no SecureChannel/Security/Sequence headers in the body.
- **TLS is the secure channel.** Confidentiality, integrity, and (optionally) mutual authentication come from the TLS layer, so OPC UA `OpenSecureChannel` is not performed.
- **No HEL/ACK on the wire.** The buffer-negotiation handshake does not happen over HTTP.

The challenge: the core `opcua-client` was written for UA-TCP and emits `HEL`, expects an `ACK`, then `OPN`, then `CreateSession`. The transport's job is to make that pipeline run unmodified over HTTP.

## How the core pipeline is satisfied

`HttpsTransport implements ClientTransportInterface`. The core drives it with `send()` / `receive()` calls. The transport branches on the frame:

| Core emits | `HttpsTransport::send()` does | `receive()` returns |
| --- | --- | --- |
| `HEL` (first call) | `encoding->fakeAcknowledge($hel)` → buffers a synthetic `ACK`. **No network.** | the buffered `ACK` |
| `MSG` / `CLO` | `encoding->encodeRequest()` → one HTTPS POST → `encoding->decodeResponse()` → buffers a re-framed `MSG` | the buffered response frame |

Two more contract methods make the core skip the parts that don't apply:

- **`isSecureChannelExternal(): true`** — the core's `openSecureChannelExternal()` branch skips `OpenSecureChannel`; TLS already provides the secure channel.
- **`createProbe(): self`** — returns a fresh transport sharing the same HTTP client, encoding, endpoint, timeout, logger, and dispatcher (used by the core for endpoint probing).

`connect()` is a **no-op** (it only throws `ConnectionException` if the transport was already `close()`d) — HTTPS is stateless, there is no socket to open. `isConnected()` is simply "not closed".

## The synthetic HEL/ACK

`fakeAcknowledge()` (in both encodings) decodes the client's `HEL` frame (`MessageHeader` + `HelloMessage`) and builds an `ACK` body locally:

```
ACK body = UInt32 0                                  (reserved / protocol)
         + UInt32 hello.getReceiveBufferSize()
         + UInt32 hello.getSendBufferSize()
         + UInt32 negotiatedMaxMessageSize           (ctor arg, default 16 MiB)
         + UInt32 negotiatedMaxChunkCount            (ctor arg, default 0 = unlimited)
```

It echoes the client's own buffer sizes (there is no server to negotiate with) and uses the strategy's configured max-message/chunk values. A non-`HEL` input, or an undecodable header/body, raises `EncodingException`.

## Frame stripping & re-framing (binary)

A UA-TCP `MSG` frame the core emits looks like:

```
[3B MessageType][1B ChunkType][4B MessageSize]   ← 8-byte UA-TCP header
[4B SecureChannelId]
[4B TokenId]
[4B SequenceNumber][4B RequestId]
[NodeId TypeId + ServiceRequestHeader + body…]   ← the bare service request
└────────────── 24-byte prefix ──────────────┘
```

- **`encodeRequest()`** validates the frame is `MSG` or `CLO` and at least 24 bytes, then returns everything **after** the 24-byte prefix — the bare service request is the HTTP POST body. (Binary §7.4.4 is essentially this strip; the secure-channel/sequence fields are meaningless when TLS is the channel.)
- **`decodeResponse()`** wraps the bare HTTP response body back into a UA-TCP `MSG` frame with a **synthetic prefix** (fixed channel id `1`, token id `1`, sequence `1`, request id `1`) so the core decoder reads it transparently — those fields are read-and-discarded because the secure channel is external.

JSON (`JsonHttpsEncoding`) does the same strip, then additionally **transcodes** the bare binary service request into the `{"TypeId":…,"Body":…}` JSON envelope via a `ServiceCodecInterface`, and reverses it on the response side. See [`ENCODINGS.md`](ENCODINGS.md).

## Receive buffering & size guard

`send()` performs the POST and buffers the decoded frame; `receive()` hands back the pending `ACK` first, then the pending response. A response larger than the configured receive buffer (`setReceiveBufferSize()`, default 65535) raises the core's `ProtocolException`. Calling `receive()` with nothing buffered raises `HttpsTransportException` ("out-of-order use") — a guard against driving the transport incorrectly.

## Dependency boundary with the core

- Hard requirement: `php-opcua/opcua-client ^4.4` — `ClientTransportInterface::createProbe()` / `isSecureChannelExternal()` and the `openSecureChannelExternal()` skip are v4.4 additions.
- Reused core types: `Transport\ClientTransportInterface`, `Protocol\MessageHeader`, `Protocol\HelloMessage`, `Encoding\BinaryDecoder` / `BinaryEncoder`, `Types\NodeId`, `Exception\ConnectionException` / `ProtocolException`.
- The package adds the HTTP/encoding machinery; it ends where the core begins — a connected `Client`.
