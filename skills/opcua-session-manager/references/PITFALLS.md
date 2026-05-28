# Pitfalls reference

Mistakes AI coding assistants frequently make with `opcua-session-manager`. Read before generating code.

## 1. Using ManagedClient in a one-shot CLI script

**Wrong**:
```php
#!/usr/bin/env php
<?php
require 'vendor/autoload.php';
$client = new \PhpOpcua\SessionManager\Client\ManagedClient();
$client->connect('opc.tcp://localhost:4840');
echo $client->read('i=2259')->getValue() . "\n";
$client->disconnect();
```

Adds IPC overhead with no benefit — the script terminates after one read, no future request can reuse the session.

**Right** — use direct `Client` for one-shots:
```php
#!/usr/bin/env php
<?php
require 'vendor/autoload.php';
$client = \PhpOpcua\Client\ClientBuilder::create()->connect('opc.tcp://localhost:4840');
echo $client->read('i=2259')->getValue() . "\n";
$client->disconnect();
```

The session-manager is for **request/response** apps where sessions need to survive between requests. CLI tools don't qualify.

## 2. Storing `$client->getSessionId()` in a database for "reattach later"

**Wrong**:
```php
$sessionId = $client->getSessionId();
DB::table('opcua_sessions')->insert(['session_id' => $sessionId, ...]);

// Later, in another request:
$client->reattach($savedSessionId);                       // method doesn't exist
```

The session ID is daemon-internal and **does not survive daemon restarts**. There is no "reattach by ID" API.

**Right**: just `connect()` with the same `(endpointUrl, security, identity)` tuple. The daemon deduplicates and reuses the matching live session if one exists.

```php
// Any request, same endpoint + same credentials → same in-memory session
$client = new ManagedClient();
$client->connect('opc.tcp://plc.example:4840', userCredentials: ['operator', getenv('PASSWORD')]);
```

## 3. Disconnecting between every request

**Wrong**:
```php
// In a controller
public function temperature(): JsonResponse {
    $client = new ManagedClient();
    $client->connect('opc.tcp://plc.example:4840');
    $value = $client->read('ns=2;s=Temp')->getValue();
    $client->disconnect();                                 // throws the session away
    return response()->json(['temperature' => $value]);
}
```

Every `disconnect()` triggers CloseSession on the daemon. The next request must re-handshake from scratch — same cost as not using the session-manager at all.

**Right** — let the session sit. The inactivity-timeout cleanup (default 600 s) reaps stale sessions automatically.

```php
public function temperature(): JsonResponse {
    $client = new ManagedClient();
    $client->connect('opc.tcp://plc.example:4840');
    $value = $client->read('ns=2;s=Temp')->getValue();
    // No disconnect() — daemon keeps the session warm
    return response()->json(['temperature' => $value]);
}
```

Disconnect only when you know the session won't be reused (end of a long-running daemon process, intentional cleanup, etc.).

## 4. Binding TCP to a non-loopback host

**Wrong**:
```bash
php bin/opcua-session-manager --socket=tcp://0.0.0.0:9876
# → InvalidArgumentException at construction
```

The loopback-only guard is intentional and cannot be disabled. Cross-host IPC is not supported.

**Right** — if you really need cross-host, use SSH port-forwarding or a Unix-socket-over-SSH tunnel. Better: run a session-manager per host and have the application connect to the local one.

## 5. Catching `DaemonException` and retrying silently

**Wrong**:
```php
for ($i = 0; $i < 3; $i++) {
    try {
        $client->connect(...);
        break;
    } catch (DaemonException $e) {
        sleep(1);
    }
}
```

`DaemonException` means: socket unreachable, auth token wrong, daemon refused the command, daemon crashed mid-call. None of these are transient — retrying won't help.

**Right**:
```php
try {
    $client->connect($url);
} catch (DaemonException $e) {
    // Hard failure — log, alert, fall back to direct Client, OR fail the request
    $this->log->error('Session manager unreachable', ['error' => $e->getMessage()]);
    throw new ServiceUnavailableException();
}
```

For framework integrations: configure `laravel-opcua` / `symfony-opcua` to fall back to direct `Client` when the daemon is unreachable — they handle the catch automatically.

## 6. Hard-coding the socket path

**Wrong**:
```php
$client = new ManagedClient(endpoint: '/tmp/opcua-session-manager.sock');
```

Breaks on Windows (TCP loopback instead of Unix socket). Breaks on production (admin moved it to `/var/run/opcua/sm.sock`).

**Right**:
```php
use PhpOpcua\SessionManager\Ipc\TransportFactory;

$client = new ManagedClient(
    endpoint: $_ENV['OPCUA_SOCKET'] ?? TransportFactory::defaultEndpoint(),
);
```

`TransportFactory::defaultEndpoint()` picks per OS. Env var override is the standard pattern for production.

## 7. Passing the auth token on the command line

**Wrong**:
```bash
php bin/opcua-session-manager --auth-token=hunter2
# anyone with shell access can `ps aux` and read it
```

**Right** — use `--auth-token-file` or `OPCUA_AUTH_TOKEN` env:
```bash
echo -n 'hunter2' > /etc/opcua/sm.token && chmod 600 /etc/opcua/sm.token && chown opcua:opcua /etc/opcua/sm.token
php bin/opcua-session-manager --auth-token-file=/etc/opcua/sm.token

# or
OPCUA_AUTH_TOKEN=hunter2 php bin/opcua-session-manager
```

## 8. Setting `setEventDispatcher` / `setLogger` on ManagedClient

**Wrong**:
```php
$client = new ManagedClient();
$client->setEventDispatcher($dispatcher);                  // refused by daemon — blocked setter
```

PSR-3 logger and PSR-14 dispatcher are **daemon-side** concerns. They configure the daemon's internal `Client` for every session.

**Right** — wire them when constructing the daemon:
```php
$daemon = new SessionManagerDaemon(
    clientEventDispatcher: $dispatcher,
    logger: $logger,
);
```

The PSR-14 events fire on the daemon side; if your application needs to react, configure a listener in your framework that's also running daemon-side (auto-publish + Laravel/Symfony integration handles this).

The client-side `ManagedClient::__construct(?LoggerInterface $logger)` parameter accepts a logger for **client-side IPC logging** (e.g. "sent connect command, waiting for ack"). It does NOT propagate to the daemon's `Client`.

## 9. Calling `setTrustStore` / `setMaxRetries` / other setters

Same as #8 — every setter is blocked by the whitelist. Configure them on the daemon's auto-connect / programmatic config, not on `ManagedClient`.

## 10. Forgetting `--allowed-cert-dirs` and trying to load certs

**Wrong**:
```php
$client->connect('opc.tcp://...', clientCertPath: '/home/user/certs/client.pem');
// → daemon refuses: cert path not in allowed dirs (default: none)
```

**Right** — set the whitelist at daemon startup:
```bash
php bin/opcua-session-manager --allowed-cert-dirs=/etc/opcua/certs
```

Then put your certs there:
```php
$client->connect('opc.tcp://...', clientCertPath: '/etc/opcua/certs/client.pem');
```

OR let the daemon auto-generate a self-signed cert (no path needed) — works for most policies.

## 11. Reading large history responses in one call

**Wrong**:
```php
$values = $client->historyReadRaw('ns=2;s=Temp', $startMonthAgo, $now, maxValues: 10_000_000);
// → SerializationException: envelope > 16 MiB
```

The IPC envelope has a 16 MiB hard cap. A million DataValues serialised typically exceeds that.

**Right** — page or chunk:
```php
$cursor = $startMonthAgo;
while ($cursor < $now) {
    $end = min($cursor->add(new DateInterval('PT1H')), $now);
    $batch = $client->historyReadRaw('ns=2;s=Temp', $cursor, $end, maxValues: 50_000);
    foreach ($batch as $dv) { /* process */ }
    $cursor = $end;
}
```

Or use `--max-request-size` to tune the daemon (rarely the right answer).

## 12. Expecting auto-publish events to arrive on the client side

**Wrong**:
```php
$client = new ManagedClient();
$client->connect(...);
$client->createSubscription(...);

// Hoping events fire on the application side:
class MyListener {
    public function onDataChange(DataChangeReceived $e): void { /* ... */ }
}
```

Auto-publish events fire on the **daemon** side, not the client side. Your listener has to run inside the daemon's PSR-14 dispatcher to receive them.

**Right** — for Laravel / Symfony, use the framework integration's auto-wiring. The framework's event bus is plumbed into the daemon's dispatcher automatically.

For raw PHP daemons that want listeners in the same process: implement them as daemon-side modules or pass a custom `EventDispatcherInterface` to `SessionManagerDaemon::__construct()`.

## 13. Mixing v4.3 client with v4.4 daemon (or vice versa)

The wire envelope is currently stable across v4.x patches BUT:

- The v4.4 daemon advertises 51 whitelisted methods including HistoryUpdate / Aggregate / FileTransfer.
- A v4.3 ManagedClient won't have typed wrappers for the new methods — but `invokeRemote('historyInsertData', [...])` works.
- A v4.3 daemon won't have the new methods in its whitelist — `historyInsertData` raises "method not allowed".

**Right** — keep daemon and client on the same minor version. Composer constraints already enforce this:

```json
"require": {
  "php-opcua/opcua-session-manager": "^4.4",
  "php-opcua/opcua-client": "^4.4"
}
```

## 14. Building a custom transport that doesn't enforce loopback

If you fork `TcpLoopbackTransport` or implement a custom TLS transport, **keep the loopback-only guard** unless you really know what you're doing. Removing it exposes the daemon's IPC to anyone with network reachability — every cached credential, every active session.

If you genuinely need cross-host IPC, use mTLS + a private CA + per-machine certificates + explicit allowlisting. Don't just remove the guard.

## 15. Trying to share the daemon socket across Docker containers without a volume

**Wrong**:
```yaml
services:
  daemon:
    command: opcua-session-manager --socket=unix:///tmp/sm.sock
  app:
    environment:
      OPCUA_SOCKET: unix:///tmp/sm.sock                   # different container's /tmp
```

Each container has its own `/tmp`. The app sees no socket.

**Right** — share a volume:
```yaml
services:
  daemon:
    volumes: [opcua-sock:/var/run/opcua]
    command: opcua-session-manager --socket=unix:///var/run/opcua/sm.sock
  app:
    volumes: [opcua-sock:/var/run/opcua:ro]
    environment:
      OPCUA_SOCKET: unix:///var/run/opcua/sm.sock

volumes:
  opcua-sock:
```

Or use `tcp://127.0.0.1:port` with `network_mode: service:daemon` on the app container (shares the daemon's network namespace).

## 16. Logging the SessionConfig without sanitization

**Wrong**:
```php
$this->log->info('Opening session', ['config' => $config->toArray()]);
// password leaks into the log
```

**Right**:
```php
$this->log->info('Opening session', ['config' => $config->sanitized()->toArray()]);
```

`SessionConfig::sanitized()` blanks `SENSITIVE_FIELDS` (`password`, `clientKeyPath`, `caCertPath`, `userKeyPath`). The CommandHandler always uses this when logging — you should too if you log config from custom code.

## 17. Not handling `ping()` properly in health checks

`ManagedClient::ping()` returns `bool`. It tests the IPC link (daemon reachable + auth accepted), NOT the OPC UA server reachability.

For a true end-to-end health check:

```php
public function healthCheck(): bool {
    $client = new ManagedClient(authToken: $_ENV['OPCUA_AUTH_TOKEN']);
    if (!$client->ping()) return false;                  // daemon down

    try {
        $client->connect('opc.tcp://known-good-server:4840');  // OPC UA layer
        return true;
    } catch (\Throwable) {
        return false;
    }
}
```
