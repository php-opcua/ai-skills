# Driving the listener

Everything about binding, accepting, timeouts, frame limits, and the factory callback.

## Constructing the listener

```php
new ReverseConnectListener(
    string $bindHost,                       // '0.0.0.0' | '127.0.0.1' | a specific NIC IP
    int $bindPort,                          // 0 = kernel-assigned ephemeral port
    ReverseHelloValidator $validator,       // required — the trust gate
    ?LoggerInterface $logger = null,        // PSR-3; defaults to NullLogger
    ?EventDispatcherInterface $dispatcher = null, // PSR-14; null = events disabled
    int $maxFrameSize = 65535,              // hard cap on a single RHE frame
);
```

`logger` and `dispatcher` are named/optional. Pass `dispatcher:` and `logger:` by name when skipping the other:

```php
new ReverseConnectListener('0.0.0.0', 0, $validator, dispatcher: $myDispatcher);
```

## Bind address — the decision that bites in CI

| Server location | Bind to | Why |
| --- | --- | --- |
| Same host, loopback | `127.0.0.1` | Fine; nothing else can reach the port. |
| Remote host / another container / behind NAT | `0.0.0.0` (or the specific reachable NIC) | The server dials an address that is **not** loopback. In Docker, `host.docker.internal` resolves to the bridge gateway IP, which cannot reach a `127.0.0.1`-bound listener. |

A wrong bind here does not error — it silently never receives the connection, so `accept()` ends in `ReverseConnectTimeoutException`. See [`PITFALLS.md`](PITFALLS.md).

## Port 0 + `getBindAddress()`

Pass `bindPort = 0` to let the kernel pick a free port (ideal for tests and ephemeral listeners), then read it back:

```php
$listener = new ReverseConnectListener('0.0.0.0', 0, $validator);
$listener->listen();
[$host, $port] = explode(':', $listener->getBindAddress());
// hand $port to whatever tells the server where to dial
```

`getBindAddress()` returns the kernel-reported `host:port` and throws `ReverseConnectException` if called before `listen()`.

## Lifecycle

```php
$listener->listen();            // bind; idempotent (second call is a no-op)
$listener->isListening();       // bool
$session = $listener->accept(timeoutSeconds: 30.0);
$listener->close();             // close server socket; idempotent, safe to repeat
```

`listen()` throws `ReverseConnectException` if the bind fails (port in use, permission, bad host).

## `accept(float $timeoutSeconds)`

One inbound connection per call. The timeout is the **total wall-clock budget** for both waiting for a connection *and* reading the full RHE frame.

Internally:
1. `stream_select()` on the server socket with the given timeout → `0` changed ⇒ `ReverseConnectTimeoutException`.
2. `stream_socket_accept()` the connection.
3. `stream_set_timeout()` on the accepted socket (≥ 1s) so a stalled peer can't hang the read forever.
4. `readFrame()` → header (8 B), then `MessageSize - 8` bytes, bounded by `maxFrameSize`.
5. Dispatch `ReverseHelloReceived`.
6. `validator->ensureAccepted()` → on rejection: close socket, dispatch `ReverseConnectRejected`, throw `ReverseConnectRejectedException`.
7. Dispatch `ReverseConnectAccepted`, return `ReverseConnectSession`.

On **any** failure after accept (parse error, rejection), the listener closes the accepted socket before throwing — only a *successful* `accept()` transfers socket ownership to the returned session.

### Exceptions from `accept()`

| Throws | Cause |
| --- | --- |
| `ReverseConnectException` | listener not started; `stream_select()` / `stream_socket_accept()` failed |
| `ReverseConnectTimeoutException` | no connection within the budget |
| `ReverseHelloParseException` | malformed/truncated/oversized frame, peer closed early, read timeout |
| `ReverseConnectRejectedException` | validator refused the (well-formed) message |

## Serving multiple servers — loop `accept()`

There is no event loop and no concurrency. Re-arm by calling `accept()` again. Keep one rejection from killing the loop:

```php
$listener->listen();
while ($running) {
    try {
        $session = $listener->accept(timeoutSeconds: 5.0);
    } catch (ReverseConnectTimeoutException) {
        continue;                       // idle tick — poll again
    } catch (ReverseConnectRejectedException | ReverseHelloParseException $e) {
        $logger->warning('RHE refused', ['error' => $e->getMessage()]);
        continue;                       // socket already closed by the listener
    }

    $client = (new ReverseConnectClientFactory())->buildClient($session, $configure);
    handle($client);                    // your logic
    $client->disconnect();
}
$listener->close();
```

For true concurrency, run one listener per process/worker, or accept-then-hand-off to a job queue. The session's socket is a plain stream resource, but it is **not** safe to share across processes after the UA handshake has started.

## `maxFrameSize`

Caps a single RHE frame. The default (65535) is generous — an RHE only carries two URIs. Lower it to harden against a hostile peer declaring a huge `MessageSize`; the listener rejects (`ReverseHelloParseException`) before allocating the body. A frame whose declared size is below `MIN_FRAME_SIZE` (16) is also rejected.

## The factory callback

```php
$client = (new ReverseConnectClientFactory())->buildClient(
    $session,
    static function (ClientBuilder $b): void {
        $b->setSecurityPolicy(SecurityPolicy::Basic256Sha256)
          ->setSecurityMode(SecurityMode::SignAndEncrypt)
          ->setLogger($logger)
          ->setEventDispatcher($dispatcher);
        // identity, trust store, custom modules — anything ClientBuilder supports
    },
    readTimeout: 10.0,   // optional; applied to the inherited socket
);
```

The callback runs **after** `setTransport()` and **before** `connect()`. Configure security exactly as you would for an ordinary outbound client — the announced `endpointUrl` and the server's certificate drive validation, independent of how the listener was bound. See [`SECURITY.md`](SECURITY.md).
