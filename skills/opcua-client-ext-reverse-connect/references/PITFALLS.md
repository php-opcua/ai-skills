# Pitfalls

Failure signatures and the fixes, ordered by how often they bite.

## 1. `accept()` times out — listener bound to the wrong interface

**Symptom:** `ReverseConnectTimeoutException: No inbound reverse-connect connection within N.000s`, even though the server reports it dialed.

**Cause:** the listener is bound to `127.0.0.1` but the server reaches the host from elsewhere — another machine, or a container where `host.docker.internal` resolves to the **bridge gateway IP**, not loopback. A loopback-bound socket is invisible on that path.

**Fix:** bind `0.0.0.0` (or the specific reachable NIC). Loopback is only correct when the server can genuinely reach `127.0.0.1` on the same host. The bind interface has no bearing on certificates — see [`SECURITY.md`](SECURITY.md).

## 2. Empty whitelist rejects everything

**Symptom:** every message → `ReverseConnectRejectedException: ... is not in the configured whitelist`.

**Cause:** `new ReverseHelloValidator([])` is **fail-secure**, not "accept all".

**Fix:** enumerate the trusted `ServerUri`s explicitly: `new ReverseHelloValidator(['urn:opcua:testserver:nodes'])`. There is no accept-all mode by design.

## 3. Whitelist miss on case / exact match

**Symptom:** a server you "added" is still rejected.

**Cause:** matching is exact and **case-sensitive** (`in_array(strict: true)`). `urn:Foo` ≠ `urn:foo`; trailing slashes and whitespace count.

**Fix:** copy the server's `ApplicationUri` verbatim. Log the rejected URI (`$e->rejectedMessage->serverUri`) and compare byte-for-byte.

## 4. First connect against a fresh server → `BadServerHalted` (`0x800E0000`)

**Symptom:** `ServiceException: Server returned ServiceFault: 0x800E0000` on the very first `connect()`, typically only in CI, passing locally.

**Cause:** a just-started OPC UA server (notably UA-.NETStandard) answers with the top-level ServiceFault `BadServerHalted` until its internal state reaches Running. Locally the server has been up for a while, so you never hit the window; in CI it's freshly booted. A premature container healthcheck (`docker compose --wait` reporting healthy too early) makes it worse.

**Fix:** retry the connect through the boot window (catch `ServiceException`/`ConnectionException`, back off ~250ms, bounded to ~15s). Server-side, gate the healthcheck on a real readiness marker (test-suite v1.5.1+). Both together remove the race. See [`TESTING.md`](TESTING.md).

> `0x800E0000` is unmapped in `StatusCode::getName()`, so the error message echoes the raw code twice (`0x800E0000 0x800E0000`). It is the standard OPC UA `BadServerHalted`.

## 5. Re-dialing the endpoint instead of reusing the socket

**Symptom:** a second outbound connection is attempted; it hangs or fails because the server is behind NAT/firewall and unreachable outbound.

**Cause:** calling `ClientBuilder::create()->connect($session->endpointUrl)` directly throws away the socket the server already handed you and opens a fresh outbound channel — exactly what Reverse Connect exists to avoid.

**Fix:** always go through `ReverseConnectClientFactory::buildClient($session, $configure)`. It wraps the live socket via `TcpTransport::fromConnectedSocket()` and the core skips the redundant connect.

## 6. Leaked sockets / listener

**Symptom:** "address already in use" on the next bind, or lingering FDs.

**Cause:** `accept()` succeeded (session owns the socket) but the resulting `Client` was never `disconnect()`ed; or the listener was never `close()`d.

**Fix:** `disconnect()` the client and `close()` the listener in `finally`. Note: on a *failed* `accept()` (parse/reject), the listener already closes the accepted socket for you — don't double-close.

## 7. Treating `accept()` as "session is up"

**Symptom:** code reads/writes immediately after `accept()` and gets a "not connected" error.

**Cause:** `accept()` only means the RHE was received and validated. The UA-TCP handshake (HEL/ACK, OPN, CreateSession) happens in `buildClient()`.

**Fix:** use the `Client` returned by the factory for all service calls, not the raw session.

## 8. Oversized / malformed frame from a hostile or buggy peer

**Symptom:** `ReverseHelloParseException` with "exceeds the configured maximum", "does not match received length", or "Trailing N byte(s)".

**Cause:** declared `MessageSize` over `maxFrameSize`, a size/length mismatch, or junk after `EndpointUrl`. The parser is strict on purpose.

**Fix:** this is correct rejection — the socket is closed and the exception thrown before any body is allocated for an oversized frame. Keep `maxFrameSize` tight. If a *legitimate* server is rejected, verify it actually emits a spec-compliant RHE (`RHE` + `F` + total-size UInt32 LE).

## 9. Peer closes mid-frame

**Symptom:** `ReverseHelloParseException: Connection closed by peer before ReverseHello frame was complete` or `Timeout while reading ReverseHello frame from peer`.

**Cause:** the server connected but didn't finish sending the framed RHE within the read budget (network blip, server crash, wrong protocol on the port).

**Fix:** confirm the peer is actually an OPC UA reverse-connecting server (not a port scanner or load-balancer health probe). The read timeout derives from the `accept()` budget; give a slow link a larger `timeoutSeconds`.

## 10. Expecting concurrency from one listener

**Symptom:** a second server's RHE is missed while you're handling the first client.

**Cause:** the listener is blocking and single-threaded — no event loop.

**Fix:** loop `accept()` and hand each session off quickly, or run one listener per worker/process. See the multi-server loop in [`LISTENER.md`](LISTENER.md).
