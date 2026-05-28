---
name: opcua-session-manager
description: Keep OPC UA sessions alive across PHP requests via a ReactPHP daemon and local IPC (Unix-domain socket on Linux/macOS, TCP loopback on Windows — auto-selected). ManagedClient is a drop-in OpcUaClientInterface replacement for php-opcua/opcua-client's Client — same API surface, persistent sessions, ~150ms handshake overhead paid once instead of every request. Use this skill whenever the user wants to eliminate per-request OPC UA connection cost, run OPC UA from PHP-FPM / Laravel / Symfony web requests, keep subscriptions / monitored items alive between requests, or operate a long-running OPC UA daemon process.
license: MIT
compatibility: Requires PHP >= 8.2 with ext-openssl. ext-pcntl recommended for signal-driven shutdown. The daemon needs to run as a long-lived process (systemd / Supervisor / Docker). Lock-step with php-opcua/opcua-client v4.4.0+.
metadata:
  package: php-opcua/opcua-session-manager
  version: v4.4.0
  ecosystem: php-opcua
---

# php-opcua/opcua-session-manager — v4.4.0 skill

A ReactPHP daemon that holds OPC UA sessions in memory across short-lived PHP requests. `ManagedClient` is a drop-in `OpcUaClientInterface` replacement that talks to the daemon via local IPC — the application code is **identical** to direct `Client` usage, but the OPC UA handshake (50–200 ms) is paid once at daemon startup, not per request.

## When to use this skill

Activate when the user is solving one of these problems:

- **OPC UA from web requests** (Laravel/Symfony/FPM): every request triggers Connect → CreateSession → ActivateSession (~150 ms). The daemon caches the session.
- **Keeping subscriptions alive** across requests so notifications aren't lost between page loads.
- **Auto-publish loop** running outside the request lifecycle, dispatching PSR-14 events to the framework's listener bus.
- **Long-lived OPC UA gateway** consuming from PLCs and forwarding to HTTP / Kafka / MQTT.
- **Cross-platform IPC** — they want it to work on Windows too (auto-falls-back to TCP loopback).

Do NOT activate for: direct connections from CLI scripts that run once (`bin/opcua-cli read ...`) — those don't benefit from the daemon. Use `opcua-client` directly.

## The 60-second mental model

```
PHP request (short-lived)                       SessionManagerDaemon (long-lived)
─────────────────────────                       ─────────────────────────────────
new ManagedClient($endpoint)                    SessionManagerDaemon::run()
       │                                                  │
       ▼                                                  ▼
SocketConnection ──IPC (NDJSON over socket)──► CommandHandler
       │                                                  │
       ▼                                                  ▼
[serialize args]                            [dispatch via 51-method whitelist]
       │                                                  │
       │                                                  ▼
       │                                          SessionStore
       │                                          │
       │                                          ▼
       │                                  Session (live Client + metadata)
       │                                          │
       │                                          ▼
       │                                  OPC UA Server (opc.tcp:// or opc.https://)
       │                                          │
       └◄────[deserialize result]◄────────────────┘
```

Three things to know:

1. **Two processes, same API**. The PHP request talks to `ManagedClient` which implements `OpcUaClientInterface`. The daemon talks to `Client` (the same interface). Application code is identical between "with daemon" and "without daemon" modes.

2. **Local IPC, never network**. By default `unix:///tmp/opcua-session-manager.sock` (Linux/macOS) or `tcp://127.0.0.1:<port>` (Windows). TCP transports are **forced to loopback** at construction — a `tcp://0.0.0.0:...` bind raises immediately, both client and daemon side.

3. **51-method whitelist**. The daemon refuses any IPC command not on the whitelist. Setters (e.g. `setEventDispatcher`) are blocked — daemon-side state belongs to the daemon, not the request. Custom-module methods register at daemon startup, then route via `invoke` IPC + a typed param-deserializer registry.

## Quick start (90% of use cases fit this shape)

### 1. Start the daemon

```bash
php bin/opcua-session-manager
# defaults: socket=/tmp/opcua-session-manager.sock, timeout=600s, cleanup=30s, max-sessions=100
```

Or with options:

```bash
php bin/opcua-session-manager \
    --socket=/var/run/opcua/sm.sock \
    --timeout=1800 \
    --max-sessions=50 \
    --auth-token-file=/etc/opcua/sm.token \
    --socket-mode=0660 \
    --allowed-cert-dirs=/etc/opcua/certs
```

In production use systemd / Supervisor / Docker — see [`references/DEPLOYMENT.md`](references/DEPLOYMENT.md).

### 2. Use ManagedClient from PHP code

```php
use PhpOpcua\SessionManager\Client\ManagedClient;

$client = new ManagedClient();                  // defaults to per-OS auto-endpoint
$client->connect('opc.tcp://localhost:4840');

// Every OpcUaClientInterface method works identically to the direct Client
$value = $client->read('i=2259');
echo $value->getValue();

$refs = $client->browse('i=85');
foreach ($refs as $ref) {
    echo "{$ref->displayName} ({$ref->nodeId})\n";
}

$client->disconnect();
```

The first `connect()` triggers a real CreateSession on the daemon (~150 ms). Subsequent `connect()` calls to the **same endpoint+credentials** reuse the in-memory session (~5 ms IPC round-trip).

### 3. Session persistence across requests

```php
// Request 1
$client = new ManagedClient();
$client->connect('opc.tcp://plc.example:4840');
// Session is now alive in the daemon. Do NOT disconnect() if you want to reuse it.

// Request 2 (any time within inactivity timeout, default 600 s)
$client = new ManagedClient();
$client->connect('opc.tcp://plc.example:4840');     // ~5 ms — session reused
$value = $client->read('ns=2;s=Temp');
```

The daemon identifies sessions by `(endpointUrl, security, identity)` tuple. Two `ManagedClient::connect()` calls from different PHP requests with matching parameters hit the same in-memory session.

## When to load deeper references

| If the task involves... | Read |
| --- | --- |
| Standing up the daemon in production (systemd, Docker, Supervisor, monitoring, log routing) | [`references/DEPLOYMENT.md`](references/DEPLOYMENT.md) |
| Configuring IPC: auth token, socket permissions, custom transport URI, Windows-specific setup | [`references/IPC.md`](references/IPC.md) |
| Auto-publish (subscriptions → PSR-14 events without a publish loop), Auto-connect (pre-declared subscriptions at startup) | [`references/AUTOMATION.md`](references/AUTOMATION.md) |
| Adding a custom ServiceModule whose methods reach through the daemon | [`references/CUSTOM-MODULES.md`](references/CUSTOM-MODULES.md) |
| Type serialization on the IPC wire: TypeSerializer + WireMessageCodec + JSON envelope | [`references/SERIALIZATION.md`](references/SERIALIZATION.md) |
| Security: 51-method whitelist, credential stripping, error sanitization, connection limits, OPCUA_AUTH_TOKEN env, allowed-cert-dirs | [`references/SECURITY.md`](references/SECURITY.md) |
| Debugging an unfamiliar error or behaviour | [`references/PITFALLS.md`](references/PITFALLS.md) |
| Complete worked examples (Laravel-FPM, Symfony Messenger worker, plain CLI tool, Docker stack) | [`assets/recipes.md`](assets/recipes.md) |

## Core API surface (must-know)

`ManagedClient` implements `PhpOpcua\Client\OpcUaClientInterface` — read the [`opcua-client` SKILL](../../../../opcua-client/.ai/skills/opcua-client/SKILL.md) for the full method surface. Anything that works on `Client` works on `ManagedClient` identically. Notable session-manager-only methods:

| Method | Purpose |
| --- | --- |
| `ManagedClient::__construct(?string $endpoint = null, ?string $authToken = null, float $connectTimeoutSeconds = 5.0, ?LoggerInterface $logger = null, ?CacheInterface $cache = null)` | `$endpoint` defaults to `TransportFactory::defaultEndpoint()` (auto OS) |
| `getSessionId(): ?string` | The daemon-side session ID assigned on `connect()` — store it across requests if you build a router |
| `getEndpointUrl(): ?string` | The active OPC UA endpoint |
| `getDaemonVersion(): string` | Round-trips to fetch `SessionManagerDaemon::VERSION` |
| `ping(): bool` | Liveness check — useful for health probes |
| `invokeRemote(string $method, array $args): mixed` | Generic IPC call — used by every method internally and reachable for custom module methods not surfaced as typed methods |

For setting up the daemon programmatically (not just via CLI):

```php
use PhpOpcua\SessionManager\Daemon\SessionManagerDaemon;

$daemon = new SessionManagerDaemon(
    socketPath: '/var/run/opcua/sm.sock',         // or tcp://127.0.0.1:9876 on Windows
    sessionTimeout: 1800.0,
    cleanupInterval: 30.0,
    maxSessions: 100,
    authToken: file_get_contents('/etc/opcua/sm.token'),
    socketMode: 0o660,
    allowedCertDirs: ['/etc/opcua/certs'],
    clientEventDispatcher: $psr14Dispatcher,       // for auto-publish PSR-14
    autoPublish: true,
    autoConnect: $autoConnectConfigs,              // pre-declared sessions
    logger: $psr3Logger,
);
$daemon->run();
```

## What v4.4.0 added on top of v4.3

The package moves in lock-step with `opcua-client` v4.4.0. Every new core method is surfaced as an **explicit typed method** on `ManagedClient` so IDE autocomplete and static analysis see them natively, even though the IPC layer routes them through the generic `invoke` command.

- **9 HistoryUpdate methods** — `historyInsertData`, `historyReplaceData`, `historyUpdateData`, `historyDeleteRawModified`, `historyDeleteAtTime`, `historyInsertEvent`, `historyReplaceEvent`, `historyUpdateEvent`, `historyDeleteEvent`
- **Aggregate** — `aggregate(DataValue[], $start, $end, $intervalMs, AggregateFunction, ?AggregateOptions)` + the `historyAggregate(NodeId|string, ...)` shortcut
- **FileTransfer (Part 5)** — Open/Read/Write/Close/Position helpers plus FileDirectoryType operations
- **Composer constraint bumped** from `^4.3.0` to `^4.4.0`
- **CI test-suite bumped** to `uanetstandard-test-suite@v1.5.0` (adds HTTPS Binary, SKS, ECC servers, plus the open62541 historizing fixture used by the new HistoryUpdate integration tests)
- **`SessionManagerDaemon::VERSION`** → `4.4.0`

No IPC protocol change — old clients continue to work against new daemons (and vice versa) so long as both speak the v4.x envelope.

## Idiomatic patterns AI agents should follow

1. **In framework integrations, use `laravel-opcua` / `symfony-opcua`** rather than instantiating `ManagedClient` directly. Their facade / autowiring picks `ManagedClient` when the daemon is reachable and falls back to direct `Client` when it isn't — same code either way.

2. **Don't call `disconnect()` between requests if you want session reuse**. Disconnect explicitly only when you're done with the session for good. The daemon's inactivity-timeout cleanup handles dangling sessions.

3. **Pass credentials through `connect()`, not setters**. `ManagedClient::connect($url, securityPolicy: ..., userCredentials: [...])` — the daemon strips credentials from in-memory state after the OPC UA session is established, so they can't be exfiltrated even if the daemon is dumped.

4. **For Windows / heterogeneous environments, let `TransportFactory` pick**. `new ManagedClient()` with no endpoint argument uses `TransportFactory::defaultEndpoint()` which picks `unix://` on Linux/macOS and `tcp://127.0.0.1:port` on Windows.

5. **Use `OPCUA_AUTH_TOKEN` env var** for the auth secret in containerised deployments — `--auth-token` on the command line is visible to anyone running `ps`.

6. **Set `--allowed-cert-dirs`** to a tight whitelist when accepting certificate paths from request payloads. Otherwise the daemon refuses to load any cert (default is no allowed dir).

7. **Enable `autoPublish` + a PSR-14 dispatcher** when running subscriptions across requests. The framework's listener system then receives `DataChangeReceived` / `AlarmActivated` / etc. without any publish-loop code in your application.

8. **Custom modules go on the daemon side**, not the client side. Register them when constructing `SessionManagerDaemon` so every session gets them. `ManagedClient::$method(...args)` routes through the generic `invoke` IPC + the registered `ParamDeserializerInterface`. See [`references/CUSTOM-MODULES.md`](references/CUSTOM-MODULES.md).

9. **Lock-step versions**. `php-opcua/opcua-session-manager` v4.4.0 requires `opcua-client` v4.4.0. Don't mix-and-match minor versions.

## Common pitfalls (read before generating code)

Don't write code that:

- Instantiates `ManagedClient` in a CLI script that runs once — no session persistence benefit, just IPC overhead. Use direct `Client`.
- Stores `$client->getSessionId()` in a long-lived database column and tries to "reattach" to it from a different request — session ID is daemon-internal, not stable across daemon restarts. Just `connect()` with the same parameters and the daemon reuses the matching session.
- Catches `DaemonException` and silently retries — that exception class is raised when the daemon socket is unreachable or the auth token is wrong. Retrying without checking is futile.
- Hard-codes `/tmp/opcua-session-manager.sock` — use `TransportFactory::defaultEndpoint()` and respect the `--socket` option / env var.
- Tries to disable the method whitelist to call something custom — register a custom module on the daemon side instead.
- Pipes the daemon log to stdout in a Docker container without a logger — `StreamLogger` is the minimal default; production uses `--logger` or pass `LoggerInterface` to the constructor.

Full catalog in [`references/PITFALLS.md`](references/PITFALLS.md).

## Related packages in the php-opcua ecosystem

- **`opcua-client`** — the core OPC UA client library. `ManagedClient` is a drop-in replacement for its `Client`.
- **`laravel-opcua`** / **`symfony-opcua`** — framework integrations. Provide `Opcua::connect()` (facade) / `OpcuaManager::connect()` (autowired). They auto-pick `ManagedClient` if the daemon is reachable, fall back to direct `Client` otherwise.
- **`opcua-cli`** — CLI tool. Includes `opcua-cli session-manager:start` / `session-manager:stop` / `session-manager:status` helpers.
- **`opcua-client-nodeset`** — pre-generated PHP types for 51 OPC Foundation companion specifications. Works transparently through `ManagedClient::loadGeneratedTypes()`.
- **`opcua-client-ext-reverse-connect`** / **`opcua-client-ext-transport-https`** — alternative wire transports. Work with the daemon: configure them in the daemon's auto-connect setup so the session-manager mediates them too.
