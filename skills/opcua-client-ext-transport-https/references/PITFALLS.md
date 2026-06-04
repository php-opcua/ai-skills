# Pitfalls

Failure signatures and fixes, ordered by how often they bite.

## 1. Setting OPC UA security over HTTPS

**Symptom:** confusion about why signing/encryption "isn't applied", or unexpected handshake behaviour.

**Cause:** TLS is the secure channel. `HttpsTransport::isSecureChannelExternal()` is `true`, so the core skips `OpenSecureChannel`; an OPC UA `SecurityPolicy`/`SecurityMode` other than `None` has no channel to apply to.

**Fix:** connect with `SecurityPolicy::None` + `SecurityMode::None` and configure security on `CurlHttpClient` (TLS) instead. See [`SECURITY.md`](SECURITY.md).

## 2. Expecting JSON to talk to a real server

**Symptom:** a JSON connection fails against UA-.NETStandard (or any stack) even though binary works.

**Cause:** §7.4.5 JSON-over-HTTPS is spec-registered but **not implemented by mainstream servers** — UA-.NETStandard rejects non-binary content types; others have no HTTPS or are binary-only.

**Fix:** use `BinaryHttpsEncoding` for live servers. Treat `JsonHttpsEncoding` as a tested reference for building codecs, not a way to reach existing servers.

## 3. `UnsupportedEncodingException` from JSON

**Symptom:** `JsonHttpsEncoding: no service codec registered for binary TypeId N` (or JSON TypeId N).

**Cause:** only `GetEndpointsCodec` ships. Any other service has no codec; also, only `ns=0` numeric TypeIds are routable.

**Fix:** implement `ServiceCodecInterface` for the service and `register()` it on the strategy. See [`ENCODINGS.md`](ENCODINGS.md).

## 4. Confusing `HttpsStatusException` with `HttpsRequestException`

**Symptom:** catching the wrong exception, or treating a 4xx/5xx as a network error.

**Cause:** two distinct layers — `HttpsRequestException` = no HTTP response at all (DNS/TLS/connect failure); `HttpsStatusException` = a real response with a non-2xx status (you get `->statusCode` and `->responseBody`).

**Fix:** catch `HttpsStatusException` to inspect the status/body (e.g. extract a fault), `HttpsRequestException` for connectivity. Both extend `HttpsTransportException` if you want one catch-all.

## 5. Disabling TLS verification against a real server

**Symptom:** "it works" in dev with `verifyTls: false`, shipped to prod.

**Cause:** `verifyTls: false` turns off cert-chain **and** hostname checks — a MITM can read/alter the OPC UA traffic.

**Fix:** keep `verifyTls: true`; for internal CAs pass a `caBundle`. `false` is for controlled tests only.

## 6. Putting CA / client-cert config on `ClientBuilder`

**Symptom:** TLS trust settings have no effect.

**Cause:** the core's certificate/trust options are for the OPC UA secure channel, which is skipped here. TLS config belongs to the HTTP client.

**Fix:** pass `caBundle` / `clientCertPath` / `clientKeyPath` to `CurlHttpClient`.

## 7. Invalid or wrong-scheme endpoint URL

**Symptom:** `HttpsTransportException: Endpoint URL must use opc.https:// or https://` or `Invalid endpoint URL`.

**Cause:** the transport normalises `opc.https://` → `https://` and rejects other schemes (e.g. `opc.tcp://`).

**Fix:** pass an `opc.https://` (or `https://`) URL to **both** the transport and `connect()`.

## 8. `EncodingException` on encode/decode

**Symptom:** `UA-TCP frame is shorter than the expected 24-byte prefix`, `encodeRequest expected MSG or CLO frame`, `HTTPS response body is empty`, or `Failed to read HEL frame header`.

**Cause:** the transport was fed a frame it didn't expect, or the server returned an empty/garbage body. Usually a server/proxy that didn't actually speak the OPC UA HTTPS mapping (e.g. an HTML error page, a redirect, a proxy interstitial).

**Fix:** confirm the endpoint is a real OPC UA HTTPS server and the proxy passes the POST through untouched. Check `HttpsStatusException`/response body first — a non-2xx often explains the "empty/garbage" body.

## 9. Reconstructing the client per request

**Symptom:** no keep-alive, a fresh TLS handshake every call, slow throughput.

**Cause:** building a new `CurlHttpClient`/`HttpsTransport`/`Client` per operation throws away the kept-alive cURL handle.

**Fix:** build one transport + client and reuse it for the session; `disconnect()` (which `close()`s the HTTP client) when done.

## 10. `receive() called without a pending response`

**Symptom:** `HttpsTransportException: out-of-order use of the transport`.

**Cause:** `receive()` was called with nothing buffered — generally only when driving the transport directly in a test, not through the core pipeline.

**Fix:** drive it through the core `Client` (which always `send()`s before `receive()`), or in a unit test enqueue a response and `send()` first.
