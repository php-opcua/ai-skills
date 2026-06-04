# Security & trust model

Reverse Connect inverts who dials, which changes the threat model. Read this before deploying a listener on a reachable interface.

## The core risk

A `ReverseConnectListener` is a **server socket open to the network**. Anyone who can reach `bindHost:bindPort` can open a TCP connection and send an `RHE` frame claiming to be any `ServerUri`. Unlike a normal client (which chooses who to dial), a reverse listener must decide whether to trust whoever shows up.

The `ReverseHelloValidator` is that decision, and it runs **before** the OPC UA secure channel opens — a cheap, defence-in-depth gate in front of (not instead of) certificate validation.

## What the validator enforces

`ensureAccepted(ReverseHelloMessage $message)` rejects, in order:

1. **Empty `ServerUri`** → `ReverseConnectRejectedException`.
2. **`ServerUri` not in the whitelist** — exact, **case-sensitive** match (`in_array(..., strict: true)`). URIs are case-sensitive per RFC 3986; `urn:foo` ≠ `urn:Foo`.
3. **Empty `EndpointUrl`**.
4. **`EndpointUrl` not starting with `opc.tcp://`** — other schemes are not routable through `TcpTransport`.

`isAccepted()` is the non-throwing boolean variant. `getAllowedServerUris()` returns the configured `list<string>`.

## Fail-secure default — empty whitelist rejects everything

```php
new ReverseHelloValidator([]);            // rejects EVERY message
new ReverseHelloValidator(['urn:a']);     // accepts only urn:a
```

This is deliberate and the opposite of what a careless reader might assume. There is **no "accept all" mode** and you should never build one. If you genuinely need to accept any announcing server in a closed lab, enumerate their URIs explicitly.

## What validation does NOT do

The whitelist authenticates the **claimed** `ServerUri` against a list — it does **not** cryptographically prove identity by itself. A hostile peer can *claim* a whitelisted `ServerUri`. The real cryptographic identity check happens in the subsequent **secure channel** (OpenSecureChannel) and **certificate validation**, which this package delegates to the core `opcua-client` via the factory's `$configure` callback.

So the layered model is:

1. **Network** — restrict who can reach the listener port (firewall, bind to a private NIC, mTLS at a proxy). The validator is not a substitute for this.
2. **Validator (this package)** — fast structural + whitelist gate; drops obvious impostors and junk before spending a secure-channel handshake on them.
3. **Secure channel + certificate trust (core)** — the cryptographic identity guarantee. Configure `SecurityPolicy`/`SecurityMode` and the trust store via `$configure`.

A production reverse listener should run **with security** (Sign or SignAndEncrypt) so step 3 actually authenticates the server. The `SecurityPolicy::None` / `SecurityMode::None` setup in the quick-start and tests is for unencrypted lab/CI use only.

## Bind interface vs. certificates — a common confusion

The listener's bind address (`0.0.0.0` vs `127.0.0.1`) is **purely a network-layer choice**: which interface accepts the inbound TCP connection. It has **no effect on certificate validation**.

Certificate validation is driven by:
- the **`EndpointUrl`** announced in the RHE (and used for the handshake),
- the **`ServerUri` / ApplicationUri**,
- the **server's certificate** (and its SAN/hostname matched against the endpoint).

None of these depend on how you bound the listener. Choosing `0.0.0.0` for reachability does not weaken or change the cert path. (Conversely, binding `127.0.0.1` does not add security — it only limits reachability to loopback.)

## Operational hardening checklist

- Bind to the **narrowest reachable interface** the server can still dial; firewall the port to the server's source address where possible.
- Keep `maxFrameSize` close to what real RHEs need; a hostile peer otherwise declares a large `MessageSize` (still rejected before allocation, but tighter is better).
- Run **with** security in production; reserve `None/None` for CI.
- Log rejections (the listener already logs at `warning` via PSR-3) and consider a PSR-14 listener on `ReverseConnectRejected` for alerting on repeated impostor attempts.
- The `ReverseConnectRejectedException` carries `->rejectedMessage` so you can record the offending `ServerUri`/`EndpointUrl` without re-parsing.

See [`PITFALLS.md`](PITFALLS.md) for the failure signatures and [`LISTENER.md`](LISTENER.md) for the `$configure` security wiring.
