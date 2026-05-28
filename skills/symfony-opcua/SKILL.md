---
name: symfony-opcua
description: Symfony 7.3+ / 8.x bundle for OPC UA. Autowires `OpcuaManager` and `OpcUaClientInterface` via DI, exposes YAML-based named connections, ships an `opcua:session` console command for the session-manager daemon, and dispatches all 56 OPC UA events through Symfony's `EventDispatcherInterface`. Use this skill whenever the user is working with OPC UA from a Symfony application — controllers, Messenger handlers, console commands, scheduled tasks, EasyAdmin / API Platform endpoints, or Pest tests.
license: MIT
version: v4.4.0
compatibility:
  php: ">= 8.2"
  symfony: "7.3.x | 7.4.x | 8.0+"
  depends_on:
    - php-opcua/opcua-client@^4.4.0
    - php-opcua/opcua-session-manager@^4.4.0
metadata:
  package: php-opcua/symfony-opcua
  packagist: https://packagist.org/packages/php-opcua/symfony-opcua
  repository: https://github.com/php-opcua/symfony-opcua
  documentation: https://php-opcua.com
  related:
    - php-opcua/opcua-client
    - php-opcua/opcua-session-manager
    - php-opcua/laravel-opcua
    - php-opcua/opcua-cli
    - php-opcua/opcua-client-nodeset
---

# symfony-opcua

A `symfony-bundle` over `php-opcua/opcua-client`. Three things to remember:

1. **No Facade, no static helpers.** Type-hint `OpcuaManager` or `OpcUaClientInterface` in your constructor. Autowiring resolves them.
2. **`OpcuaManager::connect($name)` returns the underlying `OpcUaClientInterface` directly.** Anything `opcua-client` can do, the client returned can do.
3. **Bundle uses `AbstractBundle` + `DefinitionConfigurator` + `loadExtension()`** (modern Symfony 6.1+ style). No XML/YAML service definitions to ship — wiring is code-driven.

## What this package is for

| You want to | Use |
|---|---|
| Read / write OPC UA nodes from a controller, service, command | Inject `OpcUaClientInterface` (default conn) or `OpcuaManager` |
| Talk to multiple OPC UA servers | Named connections in YAML, `$opcuaManager->connection('plc-1')` |
| Connect to a runtime-discovered endpoint | `$opcuaManager->connectTo($url, $configOverrides, as: 'cache-key')` |
| Avoid one new TCP connection per HTTP request | Run `php bin/console opcua:session` as a supervised daemon |
| React to data changes, alarms, etc. via PSR-14 → Symfony events | Configure `auto_publish: true` + per-connection `auto_connect: true`, register `#[AsEventListener]` |
| Test code that touches OPC UA without a server | `MockClient` + `self::getContainer()->set(OpcUaClientInterface::class, $mock)` |
| Stream notifications to API Platform / EasyAdmin / Mercure | Standard Symfony event listeners on `DataChangeReceived` etc. |

## Mental model

```
Controller/Service
   └── $opcuaManager (autowired)
       └── ->connection($name)
           ├── shouldUseSessionManager() == true?
           │   └── ManagedClient (IPC → daemon → TCP → server)
           │       └── TransportFactory picks UnixSocketTransport (Linux/macOS) or TcpLoopbackTransport (Windows)
           └── shouldUseSessionManager() == false?
               └── ClientBuilder::create()->...->connect()  (direct TCP, new connection per call)
```

Both branches expose the same `OpcUaClientInterface`. Your code does not know which it has.

## Quick start

```bash
composer require php-opcua/symfony-opcua

# If Flex is not enabled, register in config/bundles.php:
#   PhpOpcua\SymfonyOpcua\PhpOpcuaSymfonyOpcuaBundle::class => ['all' => true],

cp vendor/php-opcua/symfony-opcua/config/opcua.yaml config/packages/php_opcua_symfony_opcua.yaml
```

```env
# .env
OPCUA_ENDPOINT=opc.tcp://plc.example:4840
OPCUA_USERNAME=operator
OPCUA_PASSWORD=changeme
OPCUA_AUTH_TOKEN=long-random-secret
```

```php
namespace App\Controller;

use PhpOpcua\Client\OpcUaClientInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

class PlcController extends AbstractController
{
    public function __construct(
        private readonly OpcUaClientInterface $opcua,
    ) {}

    #[Route('/server-state', methods: ['GET'])]
    public function state(): JsonResponse
    {
        $state = $this->opcua->read('i=2259')->getValue();
        return $this->json(['state' => $state, 'running' => $state === 0]);
    }
}
```

## The 3 patterns you will use 90% of the time

### Pattern A — one-shot read/write (no daemon)

Best for HTTP requests, console commands, Messenger handlers. The bundle opens a TCP connection per call.

```php
public function showServerState(OpcUaClientInterface $opcua): array
{
    $state = $opcua->read('i=2259')->getValue();
    return ['state' => $state, 'running' => $state === 0];
}
```

### Pattern B — daemon-backed, transparent session reuse

When you run `php bin/console opcua:session` under systemd/supervisor, every client call goes through the daemon. Sessions are reused.

```bash
php bin/console opcua:session --timeout=600 --max-sessions=100
```

Application code is unchanged. Same `$opcua->read(...)`, now backed by `ManagedClient` automatically.

### Pattern C — `auto_publish` + Symfony event listeners

```yaml
# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    session_manager:
        auto_publish: true
    connections:
        plc-1:
            endpoint: '%env(PLC1_ENDPOINT)%'
            auto_connect: true
            subscriptions:
                - publishing_interval: 500.0
                  monitored_items:
                      - { node_id: 'ns=2;s=Temperature', client_handle: 1 }
```

```php
namespace App\EventListener;

use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class SensorReadingListener
{
    public function __invoke(DataChangeReceived $event): void
    {
        // Persist, broadcast, alert, etc.
    }
}
```

## Service-container surface

The bundle registers (all `public` so they show up in `debug:container`):

| Service | Class | What it is |
|---|---|---|
| `PhpOpcua\SymfonyOpcua\OpcuaManager` | `OpcuaManager` | The connection manager |
| `opcua` | alias of `OpcuaManager` | Backwards-compat alias |
| `PhpOpcua\Client\OpcUaClientInterface` | factory: `$opcuaManager->connection(null)` | Default connection, autowire target |
| `opcua.command.session` | `SessionCommand` | The console command (tagged `console.command`) |

```bash
php bin/console debug:container --tag=console.command | grep opcua
php bin/console debug:autowiring OpcUaClient
```

## Available console commands

| Command | Purpose |
|---|---|
| `opcua:session` | Start the session-manager daemon. Options: `--timeout`, `--cleanup-interval`, `--max-sessions`, `--socket-mode`. Other settings (log channel, cache pool, auth token, cert dirs, auto-publish) come from `session_manager` YAML config. |

The command is registered automatically when the bundle is loaded.

## Inherited methods (v4.4.0)

The bundle is a thin wrapper — every method on `OpcUaClientInterface` works through `$opcuaManager->connect()`. v4.4.0 added 21 new methods (Part 11 §6.9 HistoryUpdate, Part 5 §C.2/C.3 File Transfer, Part 13 Aggregates). They are reachable as ordinary client calls:

```php
// HistoryUpdate
$opcua->historyInsertData('ns=2;s=Backfill', $dataValues);

// File transfer
$handle = $opcua->openFile($fileNode, OpenFileMode::Read);
$bytes  = $opcua->readFile($fileNode, $handle, 65536);
$opcua->closeFile($fileNode, $handle);

// Aggregates
$bucketed = $opcua->historyAggregate('ns=2;s=Temp', $start, $end, 60000.0, AggregateFunction::Average);
```

## When to follow the references

Progressive disclosure — only load what the task needs:

- `references/CONFIG.md` — every YAML key, env vars, named connections, defaults, ServiceLocator wiring for cache pool / log channel
- `references/SESSION_MANAGER.md` — daemon command, systemd + supervisor configs, IPC endpoints, auto-publish lifecycle, mixed-version upgrade
- `references/EVENTS.md` — all 56 PSR-14 events, payload shapes, `#[AsEventListener]` patterns, Messenger-friendly handlers
- `references/INTEGRATIONS.md` — Messenger, API Platform, EasyAdmin, Mercure, FrankenPHP/Octane-style worker mode, Doctrine, Monolog channels
- `references/SECURITY.md` — policies, modes, trust store, cert auto-generation, X.509 user auth, env-driven config
- `references/TESTING.md` — Pest setup, MockClient in test container, `self::getContainer()->set(...)`, `KernelTestCase` patterns
- `references/PITFALLS.md` — common gotchas: services as request-scoped, daemon vs worker, mixed daemon versions
- `assets/recipes.md` — copy-pasteable end-to-end recipes for the most common tasks

## Idiomatic patterns

1. **Type-hint `OpcUaClientInterface` for the default connection.** Type-hint `OpcuaManager` when you need to switch between named connections.
2. **YAML defines wiring, env defines secrets.** `%env(OPCUA_PASSWORD)%` in YAML; `OPCUA_PASSWORD=...` in `.env.local`.
3. **Use named connections per server.** Don't string-build endpoints in code. Define `plc-1`, `plc-2`, `historian` in YAML.
4. **Don't disconnect in HTTP requests when the daemon is enabled.** Let the session manager handle lifecycle.
5. **For auto-published subscriptions, never call `publish()` yourself.** Returns `auto_publish_active` error.
6. **Bind Messenger handlers to a dedicated transport for OPC UA event work** so a slow PLC doesn't back-pressure the main queue.
7. **Use a dedicated Monolog channel `opcua`.** Configure `log_channel: opcua` in `session_manager`. Keeps OPC UA noise out of the main log.
8. **Run the daemon under a dedicated UID with `socket_mode: 0600`.** The web user must be in the daemon's group (or share UID).
9. **In FrankenPHP / Swoole, the bundle's `OpcuaManager` is already long-lived per worker** — just enable the daemon for transparent session reuse across requests.
10. **Override `OpcUaClientInterface` in test containers** for hermetic tests; bypass the daemon entirely.

## Exit codes (`opcua:session`)

| Code | Meaning |
|---|---|
| 0 | Daemon exited cleanly (SIGTERM/SIGINT) |
| 1 | Configuration error (invalid socket_path, missing required key) |
| 2 | Bind failure (port in use, socket-path EACCES, parent dir missing) |
| 3 | Runtime error inside daemon loop (logged via Monolog channel) |

Non-zero exits should trigger `Restart=on-failure` in systemd / `autorestart=true` in supervisor.

## Versioning

The Symfony bundle versions lock-step with `php-opcua/opcua-client` and `php-opcua/opcua-session-manager`. Upgrade order:

1. **Daemon first.** Stop `opcua:session`, `composer update`, restart.
2. **Application second.** `composer update php-opcua/symfony-opcua`.

Reverse order → `BadMethodCallException` when a v4.4 application calls a v4.4-only method against a v4.3 daemon.

## What this skill does NOT cover

- The raw OPC UA protocol — see the `opcua-client` skill.
- The session-manager daemon's IPC protocol — see the `opcua-session-manager` skill.
- CLI usage — see the `opcua-cli` skill.
- Companion-spec types (DI, IA, AutoID, etc.) — see the `opcua-client-nodeset` skill.
- Laravel patterns — see the `laravel-opcua` skill (mirror of this one for Laravel).
