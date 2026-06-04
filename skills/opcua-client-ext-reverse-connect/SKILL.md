---
name: opcua-client-ext-reverse-connect
description: Accept OPC UA Reverse Connect (ReverseHello / RHE) handshakes in PHP using php-opcua/opcua-client-ext-reverse-connect v4.4.0 — the client-side half of OPC UA Part 6 §7.1.2.3. The server dials the client; this package listens, decodes and whitelists the RHE frame, then hands a fully connected Client back through the standard ClientBuilder. Use this skill whenever a task involves Reverse Connect, ReverseHello, RHE frames, server-initiated OPC UA connections, NAT/firewall traversal for opc.tcp://, edge gateways calling home, or extending php-opcua/opcua-client with a reverse listener.
license: MIT
compatibility: Requires PHP >= 8.2, ext-openssl, and php-opcua/opcua-client ^4.4 (uses TcpTransport::fromConnectedSocket() and the ManagesConnectionTrait::performConnect() already-connected skip). Pure PHP — no C extensions.
metadata:
  package: php-opcua/opcua-client-ext-reverse-connect
  version: v4.4.0
  ecosystem: php-opcua
  extends: php-opcua/opcua-client
---

# php-opcua/opcua-client-ext-reverse-connect — v4.4.0 skill

An optional extension of [`php-opcua/opcua-client`](https://github.com/php-opcua/opcua-client) implementing the **client side** of OPC UA Reverse Connect (Part 6 §7.1.2.3). In Reverse Connect the **server initiates the TCP connection** to the client and announces itself with a ReverseHello (`RHE`) frame; from there the normal UA-TCP handshake proceeds on the same socket. This package binds the listener, decodes and validates the RHE, and bridges the live socket into the standard `ClientBuilder` flow.

Everything lives under the `PhpOpcua\Client\ExtReverseConnect\*` namespace. Applications that do not need Reverse Connect take no extra dependency.

## When to use this skill

Activate when any of these apply:

- The task mentions **Reverse Connect**, **ReverseHello**, **`RHE`**, server-initiated OPC UA connections, or "the server connects to the client"
- An OPC UA **edge gateway / PLC behind NAT or a firewall** needs to "call home" to a central client over `opc.tcp://`
- A `ReverseConnectListener`, `ReverseHelloValidator`, `ReverseConnectClientFactory`, or `ReverseConnectSession` appears in code
- The task references OPC UA Part 6 §7.1.2.3, or the `StartReverseConnect` / `StopReverseConnect` test-suite methods

Do NOT activate for: ordinary client-initiated OPC UA connections (use the core `opcua-client` skill), generic PHP/networking work, or `opc.https://` transport (use `opcua-client-ext-transport-https`).

## The 60-second mental model

```
            (server dials the client)
  OPC UA Server ───TCP connect──► ReverseConnectListener  (bind host:port)
        │                                  │ accept(timeout)  ── stream_select()
        │ sends RHE frame ────────────────►│ readFrame()
        │                                  │ ReverseHelloParser::parse()
        │                                  ▼
        │                          ReverseHelloValidator::ensureAccepted()
        │                          (ServerUri whitelist + opc.tcp:// scheme)
        │                                  ▼
        │                          ReverseConnectSession {serverUri, endpointUrl, socket}
        │                                  ▼
        └──── same socket ────────► ReverseConnectClientFactory::buildClient()
                                           │ TcpTransport::fromConnectedSocket()
                                           ▼
                                    Client (core opcua-client, fully connected)
```

Four things to know:

1. **The listener is blocking and single-threaded.** `accept(float $timeoutSeconds)` waits via `stream_select()` for one inbound connection, reads the RHE, validates it, and returns a `ReverseConnectSession`. Loop over `accept()` to service multiple servers. There is no event loop.
2. **The validator is the security boundary.** Anyone who can reach the listener port can send an RHE, so the `ServerUri` is whitelisted *before* the secure channel opens. The whitelist is **fail-secure**: empty whitelist ⇒ every message rejected.
3. **The socket is reused, not re-dialed.** The factory wraps the already-connected socket with `TcpTransport::fromConnectedSocket()`; the core's `ManagesConnectionTrait::performConnect()` detects `isConnected()` and skips the redundant outbound `connect()`, jumping straight to HEL/ACK on the inherited socket.
4. **This package speaks no OPC UA services itself.** Once `buildClient()` returns a `Client`, everything else (read/write/browse/subscribe) is the core `opcua-client` API.

## Quick start (the canonical shape)

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Client\ExtReverseConnect\ReverseConnectClientFactory;
use PhpOpcua\Client\ExtReverseConnect\ReverseConnectListener;
use PhpOpcua\Client\ExtReverseConnect\ReverseHelloValidator;
use PhpOpcua\Client\Security\SecurityMode;
use PhpOpcua\Client\Security\SecurityPolicy;

// 1. Whitelist the servers you trust (exact, case-sensitive ApplicationUri).
$validator = new ReverseHelloValidator(['urn:opcua:testserver:nodes']);

// 2. Bind a listener. Use 0.0.0.0 when the server is remote / containerised;
//    127.0.0.1 only when the server can reach loopback.
$listener = new ReverseConnectListener('0.0.0.0', 4840, $validator);
$listener->listen();

try {
    // 3. Block until a trusted server announces itself (or time out).
    $session = $listener->accept(timeoutSeconds: 30.0);

    // 4. Bridge the live socket into a standard, fully connected Client.
    $client = (new ReverseConnectClientFactory())->buildClient(
        $session,
        static function (ClientBuilder $b): void {
            $b->setSecurityPolicy(SecurityPolicy::None)
              ->setSecurityMode(SecurityMode::None);
        },
    );

    // 5. From here it's the ordinary opcua-client API.
    $value = $client->read('ns=2;s=Sensors/Temp')->getValue();

    $client->disconnect();
} finally {
    $listener->close();
}
```

## When to load deeper references

| If the task involves... | Read |
| --- | --- |
| Understanding the components, the RHE wire frame, and the socket-handoff into the core transport | [`references/ARCHITECTURE.md`](references/ARCHITECTURE.md) |
| Driving the listener: bind addresses, `accept()` timeouts, multi-server loops, frame-size limits, the factory `$configure` callback | [`references/LISTENER.md`](references/LISTENER.md) |
| The whitelist trust model, fail-secure defaults, what validation does and does *not* protect, certificates vs. bind interface | [`references/SECURITY.md`](references/SECURITY.md) |
| Writing unit tests over loopback, or the Docker-based integration suite (host.docker.internal, readiness, `StartReverseConnect`) | [`references/TESTING.md`](references/TESTING.md) |
| Debugging timeouts, `BadServerHalted`, parse errors, rejections, or unexpected behaviour | [`references/PITFALLS.md`](references/PITFALLS.md) |
| A complete working example for a specific task (single accept, multi-server loop, PSR-14 audit, secure reverse connect) | [`assets/recipes.md`](assets/recipes.md) |

## Public API surface (must-know)

All classes are in `PhpOpcua\Client\ExtReverseConnect`.

| Class | Role |
| --- | --- |
| `ReverseConnectListener` | Binds the TCP server socket; `accept()` returns a validated `ReverseConnectSession`. Ctor: `(string $bindHost, int $bindPort, ReverseHelloValidator $validator, ?LoggerInterface $logger = null, ?EventDispatcherInterface $dispatcher = null, int $maxFrameSize = 65535)`. Methods: `listen()`, `accept(float $timeoutSeconds)`, `close()`, `getBindAddress()`, `isListening()`. `listen()`/`close()` are idempotent. |
| `ReverseHelloParser` | Pure decoder. `parse(string $frame, int $maxFrameSize = 65535): ReverseHelloMessage`. Consts `MIN_FRAME_SIZE = 16`, `DEFAULT_MAX_FRAME_SIZE = 65535`. No I/O. |
| `ReverseHelloMessage` | `final readonly` DTO: `public string $serverUri`, `public string $endpointUrl`. Null OPC UA strings (length `-1`) normalised to `''`. |
| `ReverseHelloValidator` | Whitelist + scheme guard. Ctor `(iterable<string> $allowedServerUris)`. `ensureAccepted(msg)` (throws), `isAccepted(msg): bool`, `getAllowedServerUris(): list<string>`. **Empty whitelist = reject all.** |
| `ReverseConnectSession` | `final readonly` value object: `public string $serverUri`, `public string $endpointUrl`, `public mixed $socket` (live, connected TCP stream resource). |
| `ReverseConnectClientFactory` | Bridge to the core. `buildClient(ReverseConnectSession $session, ?Closure $configure = null, ?float $readTimeout = null): Client`. |

### Events (PSR-14) — dispatched only when a dispatcher is supplied to the listener

| Event | When | Payload |
| --- | --- | --- |
| `Event\ReverseHelloReceived` | after a successful decode, before validation | `public ReverseHelloMessage $message` |
| `Event\ReverseConnectAccepted` | after the validator approves, before `accept()` returns | `public ReverseHelloMessage $message` |
| `Event\ReverseConnectRejected` | when the validator refuses a syntactically valid frame | `public ReverseHelloMessage $message`, `public string $reason` |

### Exceptions — all extend `Exception\ReverseConnectException` (which extends `RuntimeException`)

| Exception | Meaning |
| --- | --- |
| `ReverseConnectException` | Base: bind failure, `accept()` before `listen()`, `stream_select()`/`stream_socket_accept()` failure. |
| `ReverseHelloParseException` | Wire-format / framing error (bad MessageType, size mismatch, truncated frame, malformed OPC UA String, trailing bytes). |
| `ReverseConnectRejectedException` | Validator rejection. Carries `public readonly ReverseHelloMessage $rejectedMessage`. |
| `ReverseConnectTimeoutException` | `accept()` budget elapsed with no inbound connection. |

## Idiomatic patterns AI agents should follow

1. **Bind `0.0.0.0` for remote/containerised servers, `127.0.0.1` only for loopback-reachable ones.** A server in a Docker container reaches the host via `host.docker.internal` → the bridge gateway IP, which cannot reach a `127.0.0.1`-bound listener. See [`references/PITFALLS.md`](references/PITFALLS.md).
2. **Always whitelist real `ServerUri`s.** Never pass an empty whitelist expecting "accept all" — it does the opposite (fail-secure). Never disable validation.
3. **Treat `accept()` as one-shot per call.** Wrap it in a loop for multiple servers; each call returns one session. There is no built-in concurrency.
4. **Reuse the socket via the factory — don't re-`connect()` to `endpointUrl` yourself.** That would open a second outbound channel and defeat the whole point of Reverse Connect (the server may be behind NAT and unreachable outbound).
5. **`close()` the listener in `finally`; `disconnect()` the client in `finally`.** The session owns the socket until the factory's transport takes it over.
6. **Logs via PSR-3 (`$logger` ctor arg), events via PSR-14 (`$dispatcher` ctor arg).** Events are not even constructed when no dispatcher is supplied — zero overhead.
7. **Match the security config on both ends.** The `$configure` callback sets `SecurityPolicy`/`SecurityMode`/identity exactly as a normal client would; the announced `endpointUrl` and the server certificate drive validation, not the listener bind interface.

## Common pitfalls (read before generating code)

Don't write code that:

- Binds `127.0.0.1` when the server lives in another host/container — the RHE never arrives and `accept()` throws `ReverseConnectTimeoutException`.
- Passes an empty `ReverseHelloValidator([])` thinking it accepts everything — it rejects everything.
- Calls `ClientBuilder::connect($session->endpointUrl)` directly instead of `factory->buildClient($session)` — re-dials outbound and ignores the inherited socket.
- Fails the test/run on the first connect against a freshly booted server (`BadServerHalted`, `0x800E0000`) instead of retrying through the boot window.
- Leaks the listener or session socket by not closing in `finally`.
- Assumes `accept()` returning means the *OPC UA session* is up — it only means the RHE was received and validated; the UA handshake happens in `buildClient()`.

Full catalog in [`references/PITFALLS.md`](references/PITFALLS.md).

## Related packages in the php-opcua ecosystem

- **`opcua-client`** — the core client this package extends. Once `buildClient()` returns, you are using its API. Load the `opcua-client` skill for read/write/browse/subscribe/history.
- **`opcua-client-ext-transport-https`** — `opc.https://` wire transport (Part 6 §7.4). Sibling extension on the same `ClientTransportInterface` seam.
- **`opcua-session-manager`** — keeps sessions alive across PHP requests; pair with reverse connect when the accepted `Client` must outlive a single request.
- **`uanetstandard-test-suite`** — Docker test servers exposing `TestServer/ReverseConnect/StartReverseConnect` / `StopReverseConnect` methods to drive the integration suite. Requires v1.4.0+ (methods) / v1.5.1+ (readiness-gated healthcheck).
