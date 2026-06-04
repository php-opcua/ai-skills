# Encoding strategies

The transport stays agnostic; the `HttpsEncodingStrategy` decides how a UA message becomes (and un-becomes) an HTTP body.

## The contract

```php
interface HttpsEncodingStrategy
{
    public function contentType(): string;                 // Content-Type request header
    public function acceptHeader(): string;                // Accept request header
    public function encodeRequest(string $uaTcpFrame): string;   // UA MSG/CLO frame → POST body
    public function decodeResponse(string $httpBody): string;    // HTTP body → UA MSG frame
    public function fakeAcknowledge(string $helFrame): string;   // HEL frame → synthetic ACK (no I/O)
}
```

Both shipped strategies share the same ctor knobs for the synthetic ACK:
`(int $negotiatedMaxMessageSize = 16 * 1024 * 1024, int $negotiatedMaxChunkCount = 0, …)`.

## Binary — `BinaryHttpsEncoding` (Part 6 §7.4.4) — production path

```php
use PhpOpcua\Client\ExtTransportHttps\Encoding\BinaryHttpsEncoding;
$encoding = new BinaryHttpsEncoding();                 // defaults are fine
$encoding = new BinaryHttpsEncoding(
    negotiatedMaxMessageSize: 64 * 1024 * 1024,
    negotiatedMaxChunkCount: 0,                          // 0 = unlimited
);
```

- `CONTENT_TYPE = 'application/octet-stream'` (both Content-Type and Accept).
- `encodeRequest()` requires a `MSG` or `CLO` frame ≥ 24 bytes and returns the bare service request (the 24-byte UA-TCP prefix is stripped). Anything else → `EncodingException`.
- `decodeResponse()` rejects an empty body, otherwise wraps the bytes in a synthetic UA-TCP `MSG` frame for the core decoder.
- **Works for any OPC UA service** — it's a transparent strip/re-wrap, so Read, Write, Browse, CreateSession, Call, etc. all flow. This is the encoding to use against real servers.

## JSON — `JsonHttpsEncoding` (Part 6 §7.4.5) — reference, GetEndpoints only

```php
use PhpOpcua\Client\ExtTransportHttps\Encoding\JsonHttpsEncoding;
$encoding = new JsonHttpsEncoding();                    // GetEndpointsCodec auto-registered
```

- `CONTENT_TYPE = 'application/opcua+uajson'`.
- Wraps each message in the envelope `{"TypeId": <NodeId>, "Body": <fields>}`, matching `Opc.Ua.JsonEncoder` 1.5.378.134 reversible-mode output (validated against `tests/Fixtures/UaNetStandard/`).
- **Only `ns=0` numeric TypeIds** are routable; anything else → `UnsupportedEncodingException`.
- Dispatches on a **codec registry**: request side keyed by the **binary** request TypeId, response side keyed by the **JSON** `TypeId.Id`. A service with no registered codec → `UnsupportedEncodingException` (message points to `ROADMAP.md`).
- Ships exactly one codec: **`GetEndpointsCodec`**. So out of the box, JSON encoding can only perform `GetEndpoints`.
- Accessors: `getJsonEncoder(): JsonEncoder`, `getJsonDecoder(): JsonDecoder`.

> Reality check: even though JSON is spec-registered, **no mainstream production server implements §7.4.5 end-to-end** (UA-.NETStandard rejects non-binary content types; other stacks have no HTTPS or are binary-only). Treat `JsonHttpsEncoding` as a tested, extensible reference — not a way to talk to existing servers today. Use binary for live connections.

### Adding a JSON service codec

Implement `ServiceCodecInterface` and `register()` it:

```php
interface ServiceCodecInterface
{
    public function binaryRequestTypeId(): int;          // ns=0 numeric, request (binary encoding)
    public function jsonRequestTypeId(): int;            // ns=0 numeric, request (JSON/data type)
    public function binaryResponseTypeId(): int;
    public function jsonResponseTypeId(): int;
    public function encodeRequestBody(BinaryDecoder $body): array;   // bare binary req → JSON Body
    public function decodeResponseBody(array $body): string;         // JSON Body → bare binary resp
}

$encoding = new JsonHttpsEncoding();
$encoding->register(new MyReadCodec());
```

`GetEndpointsCodec` is the reference implementation — binary request TypeId `428`, JSON request `426`, binary response `431`, JSON response `429`. Validate new codecs against fixtures the same way (see [`TESTING.md`](TESTING.md)).

## The JSON base-type codecs — `JsonEncoder` / `JsonDecoder`

Reversible-mode codecs for five base UA types, byte-exact with UA-.NETStandard's `Opc.Ua.JsonEncoder` 1.5.378.134:

| `JsonEncoder` | `JsonDecoder` |
| --- | --- |
| `encodeNodeId(?NodeId): null\|int\|array` | `decodeNodeId(array): NodeId` |
| `encodeVariant(?Variant): ?array` | `decodeVariant(array): Variant` |
| `encodeDataValue(?DataValue): ?array` | `decodeDataValue(array): DataValue` |
| `encodeStatusCode(int): ?int` | `decodeStatusCode(mixed): int` |
| `encodeDateTime(?DateTimeInterface): ?string` | `decodeDateTime(string): DateTimeImmutable` |

A service codec uses these to build its `Body` array, so new codecs reuse the base-type encoding rather than re-deriving JSON shapes.

## Choosing

- **Live server, any service** → `BinaryHttpsEncoding`.
- **Experimenting with §7.4.5 / building codecs / fixture-driven tests** → `JsonHttpsEncoding` (+ your codecs).
- XML-SOAP (§7.4.2) and legacy SOAP are **not implemented** — roadmap only. Don't reference them as available strategies.
