---
name: opcua-client-ext-transport-https
description: Connect a PHP OPC UA client over opc.https:// (OPC UA Part 6 §7.4) using php-opcua/opcua-client-ext-transport-https v4.4.0 — a ClientTransportInterface that exchanges each UA message as one HTTPS POST, with TLS acting as the secure channel (no OpenSecureChannel). Ships a Binary encoding (§7.4.4, production-ready) and a JSON encoding (§7.4.5, GetEndpoints-only reference). Use this skill whenever a task involves opc.https://, opc.wss://-style HTTPS OPC UA transport, HTTPS binary/JSON mappings, corporate-proxy/firewall-friendly OPC UA over 443, or wiring a custom transport into opcua-client.
license: MIT
compatibility: Requires PHP >= 8.2, ext-curl, ext-openssl, and php-opcua/opcua-client ^4.4 (the ClientTransportInterface seam — createProbe(), isSecureChannelExternal(), and the openSecureChannelExternal() skip). Pure PHP.
metadata:
  package: php-opcua/opcua-client-ext-transport-https
  version: v4.4.0
  ecosystem: php-opcua
  extends: php-opcua/opcua-client
---

# php-opcua/opcua-client-ext-transport-https — v4.4.0 skill

An optional transport extension of [`php-opcua/opcua-client`](https://github.com/php-opcua/opcua-client) implementing the OPC UA **HTTPS mappings (Part 6 §7.4)**. Each UA service message is exchanged as a single HTTPS `POST`; **TLS is the secure channel**, so the OPC UA `OpenSecureChannel` handshake is skipped entirely. The core `opcua-client` only adds two contract methods on `ClientTransportInterface`; everything else (transport, encoding strategies, HTTP client, events, exceptions) lives here under `PhpOpcua\Client\ExtTransportHttps\*`.

## When to use this skill

Activate when any of these apply:

- An endpoint URL starts with **`opc.https://`** (or a plain `https://` OPC UA endpoint)
- The task is to run OPC UA **over 443 / through a corporate proxy / firewall** where `opc.tcp://` is blocked
- The HTTPS **binary** (`application/octet-stream`) or **JSON** (`application/opcua+uajson`) mapping is mentioned (Part 6 §7.4.4 / §7.4.5)
- A `HttpsTransport`, `HttpsEncodingStrategy`, `BinaryHttpsEncoding`, `JsonHttpsEncoding`, `CurlHttpClient`, or `ServiceCodecInterface` appears in code
- Someone is wiring a non-TCP transport into `ClientBuilder::setTransport()`

Do NOT activate for: ordinary `opc.tcp://` connections (core `opcua-client`), Reverse Connect (`opcua-client-ext-reverse-connect`), or generic HTTP/REST work unrelated to OPC UA.

## The 60-second mental model

```
ClientBuilder->setTransport($httpsTransport)->connect('opc.https://…')
        │
        ▼  core Client::connect() pipeline (HEL → ACK → [OPN skipped] → CreateSession → …)
HttpsTransport (implements ClientTransportInterface)
        │  send(HEL)  → encoding->fakeAcknowledge()  → buffers a synthetic ACK (no network)
        │  send(MSG)  → encoding->encodeRequest()     → HTTP POST body
        │                         │
        │                         ▼
        │                 HttpClientInterface::post()  ── one POST per UA message
        │                  (CurlHttpClient, TLS here)
        │                         │
        │  receive()  ← encoding->decodeResponse()  ← HTTP response body
        ▼
  HttpsEncodingStrategy:  BinaryHttpsEncoding (§7.4.4)  |  JsonHttpsEncoding (§7.4.5)
```

Five things to know:

1. **One POST per UA message.** The transport is stateless at the OPC UA layer: `connect()` is a no-op, every `MSG`/`CLO` frame becomes an HTTPS POST, the response body is re-framed for the core decoder.
2. **TLS is the secure channel.** `HttpsTransport::isSecureChannelExternal()` returns `true`, so the core skips `OpenSecureChannel`. You connect with OPC UA **`SecurityPolicy::None` / `SecurityMode::None`** — security lives in the TLS layer of `CurlHttpClient`, not in OPC UA message security.
3. **HEL/ACK is synthesised locally.** Part 6 §7.4 does not carry the UA-TCP handshake on the wire. The encoding strategy's `fakeAcknowledge()` builds the ACK from the client's HEL with **no network traffic**, so the unchanged core pipeline still runs HEL → ACK → CreateSession.
4. **The encoding strategy is pluggable.** `BinaryHttpsEncoding` is production-ready for any service. `JsonHttpsEncoding` is a reference that currently ships **only the `GetEndpoints` codec** — see the limits below.
5. **TLS config goes on the HTTP client, not OPC UA.** Certificate verification, CA bundle, and mutual-TLS client cert/key are `CurlHttpClient` constructor options.

## Quick start (binary, the production path)

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Client\ExtTransportHttps\HttpsTransport;
use PhpOpcua\Client\ExtTransportHttps\Encoding\BinaryHttpsEncoding;
use PhpOpcua\Client\ExtTransportHttps\Http\CurlHttpClient;
use PhpOpcua\Client\Security\SecurityMode;
use PhpOpcua\Client\Security\SecurityPolicy;

$transport = new HttpsTransport(
    httpClient: new CurlHttpClient(verifyTls: true, caBundle: '/etc/ssl/certs/ca-bundle.crt'),
    encoding: new BinaryHttpsEncoding(),
    endpointUrl: 'opc.https://server.example:443/UA/',
    timeoutSeconds: 10.0,
);

$client = (new ClientBuilder())
    ->setSecurityPolicy(SecurityPolicy::None)   // TLS is the secure channel
    ->setSecurityMode(SecurityMode::None)
    ->setTransport($transport)
    ->connect('opc.https://server.example:443/UA/');

try {
    $state = $client->read('i=2259')->getValue();   // Server.ServerStatus.State
} finally {
    $client->disconnect();
}
```

## When to load deeper references

| If the task involves... | Read |
| --- | --- |
| How the transport works: POST-per-message, the HEL/ACK synthesis, the 24-byte frame strip, external secure channel | [`references/ARCHITECTURE.md`](references/ARCHITECTURE.md) |
| Choosing/configuring an encoding: binary vs JSON, content types, the codec registry, JSON's GetEndpoints-only limit | [`references/ENCODINGS.md`](references/ENCODINGS.md) |
| The HTTP layer: `HttpClientInterface`, `CurlHttpClient` options, connection reuse, writing a custom client | [`references/HTTP-CLIENT.md`](references/HTTP-CLIENT.md) |
| TLS trust, certificate verification, mutual TLS, why OPC UA security is `None` | [`references/SECURITY.md`](references/SECURITY.md) |
| Unit testing with `InMemoryHttpClient`, the JSON fixtures, the dotnet fixture generator, the integration E2E | [`references/TESTING.md`](references/TESTING.md) |
| Debugging errors / unexpected behaviour | [`references/PITFALLS.md`](references/PITFALLS.md) |
| A complete working example (binary connect, proxy, custom client, mTLS, JSON GetEndpoints) | [`assets/recipes.md`](assets/recipes.md) |

## Public API surface (must-know)

All classes are in `PhpOpcua\Client\ExtTransportHttps`.

| Class / interface | Role |
| --- | --- |
| `HttpsTransport` | `implements ClientTransportInterface`. Ctor: `(HttpClientInterface $httpClient, HttpsEncodingStrategy $encoding, string $endpointUrl, float $timeoutSeconds = 30.0, ?LoggerInterface $logger = null, ?EventDispatcherInterface $dispatcher = null)`. Accepts `opc.https://` or `https://` (normalises `opc.https://` → `https://`). `isSecureChannelExternal()` → `true`. |
| `Encoding\HttpsEncodingStrategy` | Interface: `contentType()`, `acceptHeader()`, `encodeRequest(string $uaTcpFrame)`, `decodeResponse(string $httpBody)`, `fakeAcknowledge(string $helFrame)`. |
| `Encoding\BinaryHttpsEncoding` | §7.4.4. `CONTENT_TYPE = 'application/octet-stream'`. Ctor: `(int $negotiatedMaxMessageSize = 16*1024*1024, int $negotiatedMaxChunkCount = 0)`. Production-ready for any service. |
| `Encoding\JsonHttpsEncoding` | §7.4.5. `CONTENT_TYPE = 'application/opcua+uajson'`. Ctor: `(int $negotiatedMaxMessageSize = 16MB, int $negotiatedMaxChunkCount = 0, JsonEncoder $encoder = new, JsonDecoder $decoder = new)`. `register(ServiceCodecInterface $codec)`. Ships `GetEndpointsCodec` by default; other services raise `UnsupportedEncodingException`. |
| `Encoding\Json\Service\ServiceCodecInterface` | `binaryRequestTypeId()`, `jsonRequestTypeId()`, `binaryResponseTypeId()`, `jsonResponseTypeId()`, `encodeRequestBody(BinaryDecoder): array`, `decodeResponseBody(array): string`. |
| `Encoding\Json\Service\GetEndpointsCodec` | The one shipped codec. Binary req TypeId `428`, JSON req `426`, binary resp `431`, JSON resp `429`. |
| `Encoding\Json\JsonEncoder` / `JsonDecoder` | Reversible-mode codecs for `NodeId`, `Variant`, `DataValue`, `StatusCode`, `DateTime`. |
| `Http\HttpClientInterface` | `post(HttpRequest $request, float $timeoutSeconds): HttpResponse`, `close()`. |
| `Http\CurlHttpClient` | Default impl (ext-curl). Ctor: `(bool $verifyTls = true, ?string $caBundle = null, ?string $clientCertPath = null, ?string $clientKeyPath = null, ?string $clientKeyPassword = null, array $extraCurlOptions = [])`. Reuses one cURL handle (keep-alive + TLS resumption). |
| `Http\HttpRequest` | `final readonly`: `url`, `body`, `contentType`, `acceptHeader`, `array $extraHeaders = []`. |
| `Http\HttpResponse` | `final readonly`: `statusCode`, `body`, `array $headers = []`. `isSuccessful()` = 2xx. |

### Events (PSR-14) — dispatched only when a dispatcher is supplied to `HttpsTransport`

| Event | Payload |
| --- | --- |
| `Event\HttpsRequestSent` | `string $url`, `string $contentType`, `int $bodyLength` |
| `Event\HttpsResponseReceived` | `string $url`, `int $statusCode`, `int $bodyLength` |
| `Event\HttpsRequestFailed` | `string $url`, `int $statusCode`, `Throwable $cause` |

### Exceptions — all extend `Exception\HttpsTransportException` (which extends `RuntimeException`)

| Exception | Meaning |
| --- | --- |
| `HttpsTransportException` | Base; also raised on an invalid endpoint URL and on out-of-order `receive()`. |
| `HttpsRequestException` | Network / TLS / connect-layer failure (no HTTP response available). |
| `HttpsStatusException` | Non-2xx HTTP response. Ctor `(string $message, int $statusCode, string $responseBody = '')`; exposes `public readonly int $statusCode`, `public readonly string $responseBody`. |
| `EncodingException` | Encode/decode failure (bad frame, empty body, malformed HEL/ACK). |
| `UnsupportedEncodingException` | JSON: no codec registered for a service, or a non-`ns=0`-numeric TypeId. |

## Idiomatic patterns AI agents should follow

1. **Connect with OPC UA security `None` over HTTPS.** TLS is the secure channel; set `SecurityPolicy::None` + `SecurityMode::None` and put trust/encryption config on `CurlHttpClient`. Mixing in an OPC UA security policy is wrong here.
2. **Use `BinaryHttpsEncoding` unless you specifically need JSON.** Binary is production-ready and works against real servers; JSON currently covers only `GetEndpoints` and no mainstream server implements §7.4.5 end-to-end.
3. **Put TLS settings on the HTTP client.** `verifyTls`, `caBundle`, and mutual-TLS `clientCertPath`/`clientKeyPath`/`clientKeyPassword` are `CurlHttpClient` ctor args — not `ClientBuilder` options.
4. **`verifyTls: false` is test-only.** Never disable certificate verification against a real server.
5. **Reuse one `CurlHttpClient`/transport per client.** The cURL handle is kept alive across POSTs for keep-alive and TLS resumption; don't reconstruct per request.
6. **Pass `opc.https://` (or `https://`) consistently** to both the transport and `connect()`. The transport normalises `opc.https://` → `https://` internally for cURL.
7. **Observability via PSR-14 / PSR-3** — pass a dispatcher and/or logger to `HttpsTransport`; events carry only URL + sizes/status (no bodies). Dispatcher exceptions are swallowed and logged, never propagated.
8. **To add a JSON service, implement `ServiceCodecInterface` and `register()` it** — don't expect arbitrary services to work over JSON out of the box.

## Common pitfalls (read before generating code)

- Setting an OPC UA `SecurityPolicy`/`SecurityMode` other than `None` over HTTPS — the secure channel is external (TLS); the core skips OPN.
- Expecting JSON to handle any service — only `GetEndpoints` ships; everything else raises `UnsupportedEncodingException`. And no common production server speaks §7.4.5 anyway.
- Disabling `verifyTls` against a real endpoint.
- Putting CA/cert config on `ClientBuilder` instead of `CurlHttpClient`.
- Treating a non-2xx response as a transport failure — that's a `HttpsStatusException` (you get the status + body), distinct from `HttpsRequestException` (no response at all).

Full catalog in [`references/PITFALLS.md`](references/PITFALLS.md).

## Related packages in the php-opcua ecosystem

- **`opcua-client`** — the core client this transport plugs into via `ClientBuilder::setTransport()`. Load its skill for read/write/browse/subscribe once connected.
- **`opcua-client-ext-reverse-connect`** — sibling extension (server-initiated `opc.tcp://` via ReverseHello) on the same `ClientTransportInterface` family.
- **`uanetstandard-test-suite`** — Docker test servers; the `opcua-https-binary` service (port 4852, v1.5.0+) backs the integration E2E.
