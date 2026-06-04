# Security & TLS

In `opc.https://`, security is provided by **TLS**, not by OPC UA message security. Getting this mental split right is the whole game.

## Why OPC UA security is `None`

`HttpsTransport::isSecureChannelExternal()` returns `true`. The core `opcua-client` reads that and **skips `OpenSecureChannel`** — there is no OPC UA secure channel, no message signing/encryption at the OPC UA layer, no application certificate exchange. Confidentiality and integrity come from the TLS tunnel underneath.

So you connect like this:

```php
(new ClientBuilder())
    ->setSecurityPolicy(SecurityPolicy::None)   // ← correct over HTTPS
    ->setSecurityMode(SecurityMode::None)
    ->setTransport($transport)
    ->connect('opc.https://…');
```

Setting any other `SecurityPolicy`/`SecurityMode` is a mistake here: the core won't open an OPC UA secure channel anyway, and you'd be asking for message security the §7.4 mapping doesn't carry.

This does **not** mean the connection is insecure — it means the security boundary moved from "OPC UA secure channel" to "TLS". Configure TLS properly and the channel is encrypted and authenticated.

## Where TLS is configured — `CurlHttpClient`

All TLS knobs live on the HTTP client constructor, not on `ClientBuilder`:

| Concern | Option |
| --- | --- |
| Verify server cert chain + hostname | `verifyTls: true` (default) |
| Trust a private/internal CA | `caBundle: '/path/to/ca.pem'` (else system CA store) |
| Mutual TLS — client cert | `clientCertPath: '/pki/client.pem'` |
| Mutual TLS — client key (+ passphrase) | `clientKeyPath`, `clientKeyPassword` |
| Anything else (cipher suite, TLS version, pinning…) | `extraCurlOptions: [...]` (applied verbatim) |

```php
new CurlHttpClient(
    verifyTls: true,
    caBundle: '/etc/pki/internal-ca.pem',
    clientCertPath: '/pki/client.pem',
    clientKeyPath: '/pki/client.key',
    clientKeyPassword: getenv('CLIENT_KEY_PASS') ?: null,
);
```

## `verifyTls: false` is test-only

Disabling verification turns off both the certificate-chain check and the hostname check. It is appropriate **only** in controlled test environments (the integration suite uses it against the self-signed test-server cert). Against any real server it defeats the entire point of TLS — a network attacker can MITM the OPC UA traffic. Never ship it.

## Mutual TLS for client authentication

Because there's no OPC UA application certificate exchange, **client authentication over HTTPS is done at the TLS layer** (mutual TLS) and/or with user credentials. For mTLS, supply the client cert/key to `CurlHttpClient`. For user identity, use the core's `ClientBuilder::setUserCredentials(...)` (the integration test connects with `admin` / `admin123`) — that travels inside the TLS tunnel as part of `ActivateSession`.

## Operational checklist

- Keep `verifyTls: true` everywhere except controlled tests; pin a `caBundle` for internal CAs rather than disabling verification.
- Prefer mTLS or strong user credentials — the OPC UA layer adds no auth of its own over HTTPS.
- Terminate at a trusted endpoint; remember a TLS-terminating proxy in the path sees plaintext OPC UA. Route proxies via `extraCurlOptions`, not by weakening verification.
- Events (`HttpsRequestSent` / `Received` / `Failed`) carry only URL + sizes/status, never bodies — safe to log; but the URL may contain host/path you consider sensitive.
- See the package `SECURITY.md` for the full policy and disclosure process.
