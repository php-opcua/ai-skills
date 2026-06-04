# The HTTP layer

The transport talks to an `HttpClientInterface`; the package ships `CurlHttpClient`. This is where TLS lives.

## The contract

```php
interface HttpClientInterface
{
    public function post(HttpRequest $request, float $timeoutSeconds): HttpResponse;
    public function close(): void;
}
```

- **Only POST** — OPC UA HTTPS uses POST exclusively.
- `post()` throws **`HttpsRequestException`** only at the network / TLS / connect layer (no HTTP response). A non-2xx HTTP response is **returned normally** as an `HttpResponse` — the transport turns that into `HttpsStatusException`. Keep that split when writing a custom client.
- `close()` releases pooled resources; must be safe to call repeatedly. The transport calls it from `HttpsTransport::close()`.

### Value objects

```php
final readonly class HttpRequest {
    public function __construct(
        public string $url,
        public string $body,
        public string $contentType,
        public string $acceptHeader,
        public array $extraHeaders = [],   // name => value, applied on top of Content-Type/Accept
    ) {}
}

final readonly class HttpResponse {
    public function __construct(
        public int $statusCode,
        public string $body,
        public array $headers = [],        // lower-cased name => value
    ) {}
    public function isSuccessful(): bool;  // 200–299
}
```

## `CurlHttpClient`

```php
use PhpOpcua\Client\ExtTransportHttps\Http\CurlHttpClient;

new CurlHttpClient(
    verifyTls: true,                       // verify cert chain + hostname (default)
    caBundle: '/etc/ssl/certs/ca.pem',     // null = system CA store
    clientCertPath: '/pki/client.pem',     // mutual TLS client cert (PEM)
    clientKeyPath: '/pki/client.key',      // mutual TLS private key (PEM)
    clientKeyPassword: 'secret',           // key passphrase, if any
    extraCurlOptions: [CURLOPT_PROXY => 'http://proxy.corp:3128'],
);
```

- **Keeps a single cURL handle** across requests (`curl_init()` once, `curl_reset()` per call), so HTTP keep-alive and TLS session resumption work without extra config. Released by `close()` / `__destruct()`.
- `verifyTls` controls both `CURLOPT_SSL_VERIFYPEER` and `CURLOPT_SSL_VERIFYHOST` (`2` when on, `0` when off).
- TLS options are applied only when non-null; `extraCurlOptions` are applied **verbatim, last**, so they can override defaults (proxy, HTTP version, custom timeouts, etc.).
- Timeouts: the `timeoutSeconds` the transport passes maps to both `CURLOPT_TIMEOUT_MS` and `CURLOPT_CONNECTTIMEOUT_MS`.
- On a cURL failure it throws `HttpsRequestException` with the cURL errno/error; otherwise returns the `HttpResponse` with the real status code (even 4xx/5xx).

## Writing a custom HTTP client

Implement `HttpClientInterface` to route through your own stack (a PSR-18 client, a Guzzle instance, an instrumented wrapper, a mock). The package does **not** ship a PSR-18 adapter — if you want PSR-18, wrap it yourself behind this interface. Honour the error contract:

```php
final class MyHttpClient implements HttpClientInterface
{
    public function post(HttpRequest $request, float $timeoutSeconds): HttpResponse
    {
        try {
            $psrResponse = $this->psr18->sendRequest(/* build from $request */);
        } catch (\Psr\Http\Client\ClientExceptionInterface $e) {
            throw new HttpsRequestException('transport failure: ' . $e->getMessage(), 0, $e);
        }
        // Return non-2xx as a normal HttpResponse — DON'T throw on status.
        return new HttpResponse(
            $psrResponse->getStatusCode(),
            (string) $psrResponse->getBody(),
            /* lower-cased headers */ [],
        );
    }

    public function close(): void { /* release pooled connections */ }
}
```

For tests, the suite ships `InMemoryHttpClient` (queue of responses/exceptions, records every request) — see [`TESTING.md`](TESTING.md).

## Proxies & corporate networks

The main reason to use this transport is reaching OPC UA servers where `opc.tcp://` is blocked but outbound `443` is allowed. Route through a proxy with `extraCurlOptions`:

```php
new CurlHttpClient(
    verifyTls: true,
    extraCurlOptions: [
        CURLOPT_PROXY => 'http://proxy.corp:3128',
        CURLOPT_PROXYAUTH => CURLAUTH_BASIC,
        CURLOPT_PROXYUSERPWD => 'user:pass',
    ],
);
```

Or add proxy/`Authorization` headers per request via `HttpRequest::$extraHeaders` from a custom client.
