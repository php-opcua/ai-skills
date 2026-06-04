# Testing

How the suite is structured and how to test code that uses the HTTPS transport.

## Layout

```
tests/
  Pest.php
  Unit/
    HttpsTransportTest.php                 # 14 — send/receive, HEL→ACK, status & request errors, probe, close
    Encoding/BinaryHttpsEncodingTest.php   # 11 — strip/re-wrap, ACK build, frame validation
    Encoding/JsonHttpsEncodingTest.php     #  6 — content type, ACK pass-through, unknown TypeId, custom codec
    Encoding/Json/JsonCodecTest.php        # 20 — 5 base types × fixture-match + round-trip
    Encoding/Json/Service/GetEndpointsCodecTest.php  # 4 — envelope encode, default omission, response decode
    Http/CurlHttpClientTest.php            #  4
    Helpers/InMemoryHttpClient.php         # programmable fake HttpClientInterface
  Integration/
    BinaryHttpsE2ETest.php                 # 1 — group('integration'), real opc.https:// round-trip
  Fixtures/UaNetStandard/                  # 23 files: 19 base-type JSON fixtures + GetEndpoints pairs (generated)
```

59 unit tests + 1 integration E2E. Pest. The `integration` group needs the Docker test server; everything else is network-free.

```bash
vendor/bin/pest --exclude-group=integration    # fast, no Docker
vendor/bin/pest --group=integration            # needs uanetstandard-test-suite (opcua-https-binary)
```

## Unit tests — `InMemoryHttpClient`, no network

Inject the fake HTTP client and pre-load its response queue. It records every `HttpRequest` it was given and replays queued `HttpResponse`s (or throws queued exceptions):

```php
use PhpOpcua\Client\ExtTransportHttps\Tests\Unit\Helpers\InMemoryHttpClient;
use PhpOpcua\Client\ExtTransportHttps\Http\HttpResponse;
use PhpOpcua\Client\ExtTransportHttps\Encoding\BinaryHttpsEncoding;
use PhpOpcua\Client\ExtTransportHttps\HttpsTransport;

$http = new InMemoryHttpClient();
$http->enqueueResponse(new HttpResponse(200, $bareResponseBody));

$transport = new HttpsTransport($http, new BinaryHttpsEncoding(), 'https://h/UA/');
$transport->send($helFrame);                 // no POST — buffers a synthetic ACK
expect($transport->receive())->toStartWith('ACK');

$transport->send($msgFrame);                 // one POST
$frame = $transport->receive();              // re-framed MSG
expect($http->sent)->toHaveCount(1);
expect($http->sent[0]->contentType)->toBe('application/octet-stream');
```

`enqueueException(new HttpsRequestException(...))` drives the failure paths (the transport dispatches `HttpsRequestFailed` and rethrows). A queued `HttpResponse(500, …)` drives the `HttpsStatusException` path. An empty queue makes the fake throw `HttpsRequestException` itself.

Key behaviours worth asserting: HEL→ACK without a POST, one POST per MSG, `HttpsStatusException` on non-2xx (with `->statusCode` / `->responseBody`), `HttpsRequestException` propagation, `receive()` out-of-order throwing `HttpsTransportException`, `createProbe()` sharing config, `close()` calling through to the HTTP client.

## JSON fixtures — generated, not hand-written

The fixtures under `tests/Fixtures/UaNetStandard/` — 19 base-type JSON files plus the `GetEndpoints` request/response pair (which adds `.json` + `.bin.b64`, 23 files total) — are emitted by `tools/json-fixture-generator/`, a standalone **dotnet 8** console app that uses `Opc.Ua.JsonEncoder` from NuGet **1.5.378.134** (the same UA-.NETStandard version `uanetstandard-test-suite` runs), so the PHP `JsonEncoder`/`JsonDecoder` are validated **byte-exact** against the reference implementation. The 19 base-type fixtures cover `NodeId`, `Variant`, `DataValue`, `StatusCode`, and `DateTime`.

Regenerate (e.g. after a version bump) without a local dotnet install:

```bash
docker run --rm -v "$(pwd):/work" -w /work/tools/json-fixture-generator \
  mcr.microsoft.com/dotnet/sdk:8.0 \
  dotnet run -- /work/tests/Fixtures/UaNetStandard
```

When you add a `ServiceCodecInterface`, generate a matching fixture pair the same way and assert your codec's envelope/decoding against it — that's how `GetEndpointsCodec` is tested.

## Integration E2E

`tests/Integration/BinaryHttpsE2ETest.php` runs a full round-trip against the **`opcua-https-binary`** service from [`uanetstandard-test-suite`](https://github.com/php-opcua/uanetstandard-test-suite) **v1.5.0+** (endpoint `opc.https://localhost:4852/UA/TestServer`). It:

1. builds `HttpsTransport(new CurlHttpClient(verifyTls: false), new BinaryHttpsEncoding(), …)`,
2. connects with `SecurityPolicy::None` / `SecurityMode::None` and `setUserCredentials('admin','admin123')`,
3. reads `i=2259` (`Server.ServerStatus.State`), expects `DataValue` value `0` (Running),
4. disconnects.

`verifyTls: false` is used because the test server presents a self-signed cert — test-only, never in production. There is no JSON integration test because no server implements §7.4.5 end-to-end.

## Style

Same ecosystem bar: strict types, full PHPDoc, no body comments, Pest, cross-platform (no Unix-only APIs). Cover both happy paths and every new error branch when adding behaviour.
