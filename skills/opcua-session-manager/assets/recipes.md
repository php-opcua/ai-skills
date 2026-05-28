# Recipes — complete working examples

Copy-pasteable snippets covering every realistic deployment shape.

## R1 — Local dev (bare metal, no auth)

```bash
# Terminal 1 — start daemon
php vendor/bin/opcua-session-manager
```

```php
// app code
use PhpOpcua\SessionManager\Client\ManagedClient;

$client = new ManagedClient();
$client->connect('opc.tcp://localhost:4840');
echo $client->read('i=2259')->getValue() . "\n";
// no disconnect — session stays warm
```

## R2 — Production with systemd + auth token

```bash
# /etc/opcua/sm.token (chmod 600 owner opcua)
$ openssl rand -hex 32 > /etc/opcua/sm.token
$ chown opcua:opcua /etc/opcua/sm.token && chmod 600 /etc/opcua/sm.token
```

```ini
# /etc/systemd/system/opcua-session-manager.service
[Unit]
Description=OPC UA Session Manager
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=opcua
Group=opcua
WorkingDirectory=/srv/opcua-app
ExecStart=/usr/bin/php /srv/opcua-app/vendor/bin/opcua-session-manager \
    --socket=/var/run/opcua/sm.sock \
    --timeout=1800 \
    --auth-token-file=/etc/opcua/sm.token \
    --max-sessions=200 \
    --socket-mode=0660 \
    --allowed-cert-dirs=/etc/opcua/certs
KillSignal=SIGTERM
TimeoutStopSec=30
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now opcua-session-manager
journalctl -u opcua-session-manager -f
```

```php
// app code
use PhpOpcua\SessionManager\Client\ManagedClient;

$client = new ManagedClient(
    endpoint: 'unix:///var/run/opcua/sm.sock',
    authToken: trim(file_get_contents('/etc/opcua/sm.token')),
);
$client->connect('opc.tcp://plc.example:4840',
    userCredentials: ['operator', $_ENV['PLC_PASSWORD']]);
```

## R3 — Laravel-FPM (cross-request session reuse)

```bash
# config/opcua.php published by laravel-opcua
return [
    'mode' => 'managed',                              // → ManagedClient when daemon reachable, direct Client otherwise
    'socket' => env('OPCUA_SOCKET'),
    'auth_token' => env('OPCUA_AUTH_TOKEN'),
    'connections' => [
        'plc1' => [
            'endpoint' => env('PLC1_ENDPOINT'),
            'security_policy' => 'Basic256Sha256',
            'security_mode' => 'SignAndEncrypt',
            'user' => env('PLC1_USER'),
            'password' => env('PLC1_PASSWORD'),
        ],
    ],
];
```

```php
// app code (controller)
use PhpOpcua\LaravelOpcua\Facades\Opcua;

class TemperatureController extends Controller {
    public function show() {
        $client = Opcua::connect('plc1');             // daemon-mediated when available
        $temp = $client->read('ns=2;s=Temp')->getValue();
        return response()->json(['temperature' => $temp]);
        // no disconnect — daemon keeps session warm across requests
    }
}
```

The daemon is started by `php artisan opcua:session`. On the first FPM worker request the connection handshake happens (~150 ms); subsequent requests reuse it (~5 ms each).

## R4 — Symfony Messenger worker (long-lived consumer)

```yaml
# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    mode: managed
    socket: '%env(OPCUA_SOCKET)%'
    auth_token: '%env(OPCUA_AUTH_TOKEN)%'
    connections:
        historian:
            endpoint: '%env(HISTORIAN_ENDPOINT)%'
            security_policy: Basic256Sha256
            security_mode: SignAndEncrypt
            user: '%env(HISTORIAN_USER)%'
            password: '%env(HISTORIAN_PASSWORD)%'
```

```php
namespace App\MessageHandler;

use PhpOpcua\SymfonyOpcua\OpcuaManager;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class HistoryQueryHandler {
    public function __construct(private OpcuaManager $opcua) {}

    public function __invoke(HistoryQuery $query): array {
        $client = $this->opcua->connect('historian');
        return $client->historyReadRaw(
            $query->nodeId,
            $query->start,
            $query->end,
        );
    }
}
```

## R5 — Docker Compose with daemon + app

```yaml
# docker-compose.yml
services:
  opcua-sm:
    image: php:8.4-cli-alpine
    volumes:
      - opcua-sock:/var/run/opcua
      - ./certs:/etc/opcua/certs:ro
      - ./vendor:/srv/app/vendor:ro
      - ./bin:/srv/app/bin:ro
    secrets:
      - sm_token
    command: >
      php /srv/app/vendor/bin/opcua-session-manager
        --socket=unix:///var/run/opcua/sm.sock
        --socket-mode=0660
        --auth-token-file=/run/secrets/sm_token
        --allowed-cert-dirs=/etc/opcua/certs
        --max-sessions=200
        --timeout=1800
    healthcheck:
      test: ["CMD-SHELL", "php -r 'exit((new \\PhpOpcua\\SessionManager\\Client\\ManagedClient(endpoint: \"unix:///var/run/opcua/sm.sock\", authToken: file_get_contents(\"/run/secrets/sm_token\")))->ping() ? 0 : 1);'"]
      interval: 10s

  app:
    build: .
    depends_on:
      opcua-sm:
        condition: service_healthy
    volumes:
      - opcua-sock:/var/run/opcua:ro
    secrets:
      - sm_token
    environment:
      OPCUA_SOCKET: unix:///var/run/opcua/sm.sock
      OPCUA_AUTH_TOKEN_FILE: /run/secrets/sm_token

volumes:
  opcua-sock:

secrets:
  sm_token:
    file: ./secrets/sm.token
```

```php
// app code
$client = new ManagedClient(
    endpoint: $_ENV['OPCUA_SOCKET'],
    authToken: trim(file_get_contents($_ENV['OPCUA_AUTH_TOKEN_FILE'])),
);
```

## R6 — Windows (TCP loopback default)

```bash
:: cmd / PowerShell
php vendor\bin\opcua-session-manager
:: defaults to tcp://127.0.0.1:<auto-port>
```

The daemon prints its endpoint at startup. Either copy it into `OPCUA_SOCKET` env or pin a port:

```bash
php vendor\bin\opcua-session-manager --socket=tcp://127.0.0.1:9876
```

```php
$client = new ManagedClient(endpoint: 'tcp://127.0.0.1:9876');
$client->connect('opc.tcp://10.0.0.5:4840');
```

`TransportFactory::defaultEndpoint()` does the right thing per OS — passing no `endpoint` works on Windows too.

## R7 — Auto-connect + auto-publish (decl-only architecture)

```yaml
# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    mode: managed
    connections:
        plant_floor:
            endpoint: 'opc.tcp://plc.example:4840'
            user: 'operator'
            password: '%env(PLC_PASSWORD)%'
            auto_connect: true                          # daemon opens at boot
            subscriptions:
                - publishing_interval: 500
                  monitored_items:
                      - { node_id: 'ns=2;s=Plant/Temperature' }
                      - { node_id: 'ns=2;s=Plant/Pressure' }
                      - { node_id: 'ns=2;s=Plant/AlarmActive', sampling_interval: 0 }
```

```php
namespace App\EventListener;

use PhpOpcua\Client\Event\DataChangeReceived;
use PhpOpcua\Client\Event\AlarmActivated;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
class TelemetryRouter {
    public function __invoke(DataChangeReceived $e): void {
        $this->kafka->publish('telemetry', [
            'node' => (string) $e->nodeId,
            'value' => $e->dataValue->getValue(),
            'ts' => $e->dataValue->sourceTimestamp,
        ]);
    }
}

#[AsEventListener]
class AlarmRouter {
    public function __invoke(AlarmActivated $e): void {
        $this->notify->send(/* ... */);
    }
}
```

Application code never calls `connect()`, `publish()`, or `subscribe()`. The daemon does it via the config; events fire into Symfony's event bus; listeners process them. Same shape works in Laravel via `php artisan opcua:session --auto-publish`.

## R8 — Health check endpoint

```php
use PhpOpcua\SessionManager\Client\ManagedClient;
use PhpOpcua\SessionManager\Exception\DaemonException;

class HealthController {
    public function check(): JsonResponse {
        $checks = ['daemon' => false, 'opcua' => false];

        try {
            $client = new ManagedClient(
                authToken: $_ENV['OPCUA_AUTH_TOKEN'],
            );
            $checks['daemon'] = $client->ping();

            $client->connect('opc.tcp://canary-server:4840');
            $client->read('i=2259');
            $checks['opcua'] = true;
        } catch (\Throwable $e) {
            // log
        }

        $ok = $checks['daemon'] && $checks['opcua'];
        return response()->json($checks, $ok ? 200 : 503);
    }
}
```

## R9 — Custom module reaching through the daemon

See `references/CUSTOM-MODULES.md` for the full pattern. The 90-second sketch:

```php
// 1. Define module + result DTO
final class StatsModule extends \PhpOpcua\Client\Module\ServiceModule {
    public function name(): string { return 'stats'; }
    public function requires(): array { return []; }
    public function register($kernel, $session): array {
        return ['statsDump' => fn (): StatsResult => new StatsResult(/* ... */)];
    }
    public function registerWireTypes(\PhpOpcua\Client\Wire\WireTypeRegistry $r): void {
        $r->register(StatsResult::class);
    }
}

// 2. Wire on daemon
$daemon = new SessionManagerDaemon(
    socketPath: '/var/run/opcua/sm.sock',
    clientModules: [new StatsModule()],
);

// 3. Use from client (call as if it were built-in)
$client = new ManagedClient();
$client->connect('opc.tcp://localhost:4840');
$stats = $client->statsDump();                       // routes through invoke IPC
echo $stats->totalReads;
```

## R10 — Reading 1 hour of history in chunks (avoid 16 MiB cap)

```php
$end = new DateTimeImmutable();
$start = $end->sub(new DateInterval('PT1H'));
$cursor = $start;

while ($cursor < $end) {
    $batchEnd = min($cursor->add(new DateInterval('PT5M')), $end);
    $batch = $client->historyReadRaw(
        'ns=2;s=Plant/Temp',
        startTime: $cursor,
        endTime: $batchEnd,
        maxValues: 50_000,
    );

    foreach ($batch as $dv) {
        printf("[%s] %.2f\n", $dv->sourceTimestamp->format('H:i:s.v'), $dv->getValue());
    }

    $cursor = $batchEnd;
}
```

## R11 — v4.4.0 HistoryUpdate via daemon

```php
use PhpOpcua\Client\Types\BuiltinType;
use PhpOpcua\Client\Types\DataValue;
use PhpOpcua\Client\Types\Variant;

$values = [
    new DataValue(new Variant(BuiltinType::Double, 22.1), sourceTimestamp: new DateTimeImmutable('-30 minutes')),
    new DataValue(new Variant(BuiltinType::Double, 22.3), sourceTimestamp: new DateTimeImmutable('-20 minutes')),
];

$statuses = $client->historyInsertData('ns=2;s=Plant/Temp', $values);
// $statuses is int[] — per-value status code
```

Same shape works for `historyReplaceData`, `historyUpdateData`, `historyDeleteRawModified`, `historyDeleteAtTime`, plus the Event flavours.

## R12 — Aggregate via daemon

```php
use PhpOpcua\Client\Module\Aggregate\AggregateFunction;

$intervals = $client->historyAggregate(
    'ns=2;s=Plant/Temp',
    start: new DateTimeImmutable('-1 hour'),
    end: new DateTimeImmutable(),
    intervalMs: 60_000,
    function: AggregateFunction::Average,
);

foreach ($intervals as $bucket) {
    echo "[{$bucket->startTime->format('H:i')}] avg={$bucket->dataValue->getValue()}\n";
}
```

The aggregate is computed inside the daemon's `Client` (client-side aggregation per Part 13). No server round-trip beyond the raw `historyReadRaw` that feeds it.
