# Testing

How the suite is structured and how to write tests against the listener.

## Layout

```
tests/
  Pest.php
  Unit/
    ReverseHelloParserTest.php          # 13 — every structural rule
    ReverseHelloValidatorTest.php       # 12 — whitelist + fail-secure + case-sensitivity
    ReverseConnectListenerTest.php      # 15 — accept happy-path, reject, parse-fail, timeout, bind-error
    ReverseConnectClientFactoryTest.php #  4 — transport wiring + $configure contract
    Helpers/InMemoryEventDispatcher.php # local PSR-14 test fixture (copy of the core's)
  Integration/
    ReverseConnectE2ETest.php           # 4 — group('integration'), against the Docker test-suite
```

Pest. The `integration` group is excluded by default in CI's unit job and run separately with the Docker servers up.

```bash
vendor/bin/pest --exclude-group=integration   # fast, no Docker, portable
vendor/bin/pest --group=integration           # needs the test-suite containers
```

## Unit tests — loopback sockets, no OPC UA

The listener tests exercise real sockets but stay self-contained: they bind `127.0.0.1:0`, then a helper opens a client connection to the bound port and writes a crafted RHE frame. **Loopback TCP is used deliberately instead of `stream_socket_pair(STREAM_PF_UNIX, …)`** so the suite runs identically on Linux, macOS, and Windows.

Pattern for a happy-path listener test:

```php
$listener = new ReverseConnectListener('127.0.0.1', 0, new ReverseHelloValidator(['urn:srv']));
$listener->listen();
[, $port] = explode(':', $listener->getBindAddress());

// peer side: connect and write a valid RHE frame
$peer = stream_socket_client("tcp://127.0.0.1:{$port}");
fwrite($peer, buildRheFrame('urn:srv', 'opc.tcp://127.0.0.1:4840'));

$session = $listener->accept(timeoutSeconds: 2.0);
expect($session->serverUri)->toBe('urn:srv');

fclose($peer);
$listener->close();
```

Build RHE frames by hand to test the parser:

```php
function buildRheFrame(string $serverUri, string $endpointUrl): string {
    $body = packOpcUaString($serverUri) . packOpcUaString($endpointUrl);
    $size = 8 + strlen($body);
    return 'RHE' . 'F' . pack('V', $size) . $body;
}
function packOpcUaString(string $s): string {
    return $s === '' ? pack('l', -1) : pack('V', strlen($s)) . $s; // -1 = null string
}
```

Parser rules worth covering (all throw `ReverseHelloParseException`): wrong MessageType, wrong ChunkType, `MessageSize` < 16, `MessageSize` > `maxFrameSize`, `MessageSize` ≠ received length, malformed OPC UA String, trailing bytes after `EndpointUrl`.

## Events in tests

Use `Tests\Unit\Helpers\InMemoryEventDispatcher` to assert the dispatch sequence:

```php
$dispatcher = new InMemoryEventDispatcher();
$listener = new ReverseConnectListener('127.0.0.1', 0, $validator, dispatcher: $dispatcher);
// ... accept ...
expect($dispatcher->hasEvent(ReverseHelloReceived::class))->toBeTrue();
expect($dispatcher->hasEvent(ReverseConnectAccepted::class))->toBeTrue();
```

On a rejection path you'd instead assert `ReverseHelloReceived` + `ReverseConnectRejected`.

## Integration suite — against UA-.NETStandard

`tests/Integration/ReverseConnectE2ETest.php` drives a real UA-.NETStandard server from the [`uanetstandard-test-suite`](https://github.com/php-opcua/uanetstandard-test-suite) Docker action (service `opcua-no-security`, port 4840). Each test:

1. Opens a local listener on `0.0.0.0:0`.
2. Connects normally to the server at `opc.tcp://localhost:4840/UA/TestServer`.
3. Calls `TestServer/ReverseConnect/StartReverseConnect(host, port)` — the server's `ReverseConnectManager` then dials back.
4. Awaits the inbound RHE on the listener.
5. Cleans up via `StopReverseConnect`.

### Two CI-specific gotchas baked into the suite

- **Bind `0.0.0.0`, dial `host.docker.internal`.** The server runs in a container; `host.docker.internal` maps (via `extra_hosts: host-gateway`) to the Docker bridge gateway IP, which only reaches a `0.0.0.0`-bound listener — **never** `127.0.0.1`. All connecting tests bind `0.0.0.0`. (The pure-timeout test that never connects can stay on loopback.)
- **Retry the trigger `connect()` through the boot window.** A freshly started UA-.NETStandard server briefly returns a top-level ServiceFault `BadServerHalted` (`0x800E0000`) — or refuses TCP — until its `ServerInternal` reaches the Running state. `rcConnectTriggerClient()` retries for up to 15s on `ServiceException`/`ConnectionException`. The test-suite v1.5.1+ healthcheck also gates `docker compose --wait` on a real readiness marker, so the two together remove the race.

### Requirements

- `uanetstandard-test-suite` **v1.4.0+** for the `StartReverseConnect` / `StopReverseConnect` method nodes.
- **v1.5.1+** recommended for the readiness-gated healthcheck (otherwise rely on the in-test connect retry).
- The `opcua-no-security` service must carry `extra_hosts: ["host.docker.internal:host-gateway"]` (present from v1.4.0).

## Coverage

The suite targets the high coverage bar of the php-opcua ecosystem (no body comments, full PHPDoc, strict types, Pest). When adding behaviour, cover both the happy path and every new rejection/error branch, and keep unit tests Docker-free and OS-portable.
