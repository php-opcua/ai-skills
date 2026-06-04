# Recipes — complete working examples

Copy-pasteable snippets for `opc.https://`. Every recipe wires a `HttpsTransport` into `ClientBuilder`; once connected, it's the ordinary `opcua-client` API. All connect with OPC UA `SecurityPolicy::None` / `SecurityMode::None` because **TLS is the secure channel**.

## R1 — Connect over HTTPS (binary), read, disconnect

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Client\ExtTransportHttps\Encoding\BinaryHttpsEncoding;
use PhpOpcua\Client\ExtTransportHttps\Http\CurlHttpClient;
use PhpOpcua\Client\ExtTransportHttps\HttpsTransport;
use PhpOpcua\Client\Security\SecurityMode;
use PhpOpcua\Client\Security\SecurityPolicy;

$endpoint = 'opc.https://server.example:443/UA/';

$transport = new HttpsTransport(
    httpClient: new CurlHttpClient(verifyTls: true, caBundle: '/etc/ssl/certs/ca-bundle.crt'),
    encoding: new BinaryHttpsEncoding(),
    endpointUrl: $endpoint,
    timeoutSeconds: 10.0,
);

$client = (new ClientBuilder())
    ->setSecurityPolicy(SecurityPolicy::None)
    ->setSecurityMode(SecurityMode::None)
    ->setTransport($transport)
    ->connect($endpoint);

try {
    echo 'State: ' . $client->read('i=2259')->getValue() . "\n";   // Server.ServerStatus.State
} finally {
    $client->disconnect();   // also closes the underlying HTTP client
}
```

## R2 — Through a corporate proxy (OPC UA over 443 where opc.tcp is blocked)

```php
use PhpOpcua\Client\ExtTransportHttps\Http\CurlHttpClient;

$http = new CurlHttpClient(
    verifyTls: true,
    extraCurlOptions: [
        CURLOPT_PROXY       => 'http://proxy.corp:3128',
        CURLOPT_PROXYAUTH   => CURLAUTH_BASIC,
        CURLOPT_PROXYUSERPWD => 'user:pass',
    ],
);

$transport = new HttpsTransport($http, new BinaryHttpsEncoding(), 'opc.https://server.example:443/UA/', 15.0);
// ... wire into ClientBuilder as in R1
```

## R3 — Mutual TLS + OPC UA user credentials

```php
use PhpOpcua\Client\ExtTransportHttps\Http\CurlHttpClient;

$http = new CurlHttpClient(
    verifyTls: true,
    caBundle: '/etc/pki/internal-ca.pem',
    clientCertPath: '/pki/client.pem',
    clientKeyPath: '/pki/client.key',
    clientKeyPassword: getenv('CLIENT_KEY_PASS') ?: null,
);

$transport = new HttpsTransport($http, new BinaryHttpsEncoding(), 'opc.https://server.example:443/UA/');

$client = (new ClientBuilder())
    ->setSecurityPolicy(SecurityPolicy::None)
    ->setSecurityMode(SecurityMode::None)
    ->setTransport($transport)
    ->setUserCredentials('admin', 'admin123')   // identity travels inside the TLS tunnel
    ->connect('opc.https://server.example:443/UA/');
```

## R4 — Observability with PSR-3 + PSR-14

```php
$transport = new HttpsTransport(
    httpClient: new CurlHttpClient(verifyTls: true),
    encoding: new BinaryHttpsEncoding(),
    endpointUrl: 'opc.https://server.example:443/UA/',
    timeoutSeconds: 10.0,
    logger: $psr3Logger,
    dispatcher: $psr14Dispatcher,
);

// Listen for the three events (URL + sizes/status only — no bodies):
use PhpOpcua\Client\ExtTransportHttps\Event\HttpsRequestSent;
use PhpOpcua\Client\ExtTransportHttps\Event\HttpsResponseReceived;
use PhpOpcua\Client\ExtTransportHttps\Event\HttpsRequestFailed;

$psr14Dispatcher->listen(HttpsRequestSent::class, fn ($e) =>
    $metrics->timing('opcua.https.req_bytes', $e->bodyLength, ['url' => $e->url]));
$psr14Dispatcher->listen(HttpsResponseReceived::class, fn ($e) =>
    $metrics->increment('opcua.https.resp', ['status' => $e->statusCode]));
$psr14Dispatcher->listen(HttpsRequestFailed::class, fn ($e) =>
    $log->error('HTTPS failed', ['url' => $e->url, 'status' => $e->statusCode, 'err' => $e->cause->getMessage()]));
```

## R5 — Distinguish transport vs status errors

```php
use PhpOpcua\Client\ExtTransportHttps\Exception\HttpsRequestException;
use PhpOpcua\Client\ExtTransportHttps\Exception\HttpsStatusException;

try {
    $client = (new ClientBuilder())
        ->setSecurityPolicy(SecurityPolicy::None)
        ->setSecurityMode(SecurityMode::None)
        ->setTransport($transport)
        ->connect($endpoint);
} catch (HttpsStatusException $e) {
    // got an HTTP response, just not 2xx
    fwrite(STDERR, "HTTP {$e->statusCode}: {$e->responseBody}\n");
} catch (HttpsRequestException $e) {
    // no response at all — DNS / TLS / connect failure
    fwrite(STDERR, "transport failure: {$e->getMessage()}\n");
}
```

## R6 — JSON encoding: GetEndpoints (reference only)

JSON ships only the `GetEndpoints` codec, and no mainstream server implements §7.4.5 — this is for fixture-driven work and codec development, not live servers.

```php
use PhpOpcua\Client\ExtTransportHttps\Encoding\JsonHttpsEncoding;

$encoding = new JsonHttpsEncoding();          // GetEndpointsCodec auto-registered
echo $encoding->contentType();                // application/opcua+uajson

// Add your own service codec:
$encoding->register(new MyReadServiceCodec()); // implements ServiceCodecInterface

$transport = new HttpsTransport(new CurlHttpClient(verifyTls: true), $encoding, 'opc.https://h/UA/');
```

## R7 — Custom HTTP client (wrap your own stack)

```php
use PhpOpcua\Client\ExtTransportHttps\Http\HttpClientInterface;
use PhpOpcua\Client\ExtTransportHttps\Http\HttpRequest;
use PhpOpcua\Client\ExtTransportHttps\Http\HttpResponse;
use PhpOpcua\Client\ExtTransportHttps\Exception\HttpsRequestException;

final class GuzzleHttpClient implements HttpClientInterface
{
    public function __construct(private \GuzzleHttp\Client $guzzle) {}

    public function post(HttpRequest $request, float $timeoutSeconds): HttpResponse
    {
        try {
            $r = $this->guzzle->post($request->url, [
                'body'    => $request->body,
                'headers' => ['Content-Type' => $request->contentType, 'Accept' => $request->acceptHeader] + $request->extraHeaders,
                'timeout' => $timeoutSeconds,
                'http_errors' => false,                 // return non-2xx as a normal response
            ]);
        } catch (\GuzzleHttp\Exception\TransferException $e) {
            throw new HttpsRequestException('transport failure: ' . $e->getMessage(), 0, $e);
        }

        $headers = [];
        foreach ($r->getHeaders() as $name => $values) {
            $headers[strtolower($name)] = end($values);
        }

        return new HttpResponse($r->getStatusCode(), (string) $r->getBody(), $headers);
    }

    public function close(): void {}
}
```
