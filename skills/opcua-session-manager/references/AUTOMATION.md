# Automation reference

The daemon ships two automation hooks that turn the session manager from a passive proxy into an active gateway: **auto-publish** (subscriptions emit PSR-14 events without a manual loop) and **auto-connect** (pre-declared sessions / subscriptions started at daemon startup).

## Auto-publish

### Problem it solves

Plain OPC UA subscriptions require an explicit `publish()` loop:

```php
while (true) {
    $response = $client->publish();
    foreach ($response->notifications as $n) { /* ... */ }
}
```

In a PHP request/response model that loop has no home. The daemon already has one — `AutoPublisher` runs it per-session, self-rescheduling against React's event loop, and dispatches PSR-14 events to the framework's listener bus.

### Wiring

```php
use PhpOpcua\SessionManager\Daemon\SessionManagerDaemon;

$dispatcher = /* any PSR-14 EventDispatcherInterface */;

$daemon = new SessionManagerDaemon(
    socketPath: '/var/run/opcua/sm.sock',
    clientEventDispatcher: $dispatcher,            // forwarded into every Client
    autoPublish: true,
);
$daemon->run();
```

When `autoPublish: true` and a dispatcher is set:

- Every session created via the IPC `open` command **and** that subsequently calls `createSubscription` enters an internal AutoPublisher track.
- The AutoPublisher schedules a timer at `revisedPublishingInterval` (per subscription).
- On each tick the daemon calls `$client->publish()`, decodes notifications, and dispatches PSR-14 events:
  - `DataChangeReceived($client, $monitoredItemId, DataValue $dv)`
  - `EventNotificationReceived($client, $monitoredItemId, Variant[] $fields)`
  - `AlarmActivated`, `AlarmDeactivated`, `AlarmEventReceived` (auto-deduced from fields)
  - `SubscriptionKeepAlive` (when the server returns no notifications)
- Acks for previous sequence numbers are managed internally — no manual ack handling in your code.

### Framework wiring

#### Laravel

`laravel-opcua` automatically wires Laravel's event bus into the daemon when `autoPublish: true` is set in `config/opcua.php`. Listeners:

```php
use PhpOpcua\Client\Event\DataChangeReceived;

class TemperatureMonitor {
    public function handle(DataChangeReceived $event): void {
        if ($event->dataValue->getValue() > 90) {
            Notification::route('slack', '#alerts')->notify(new HighTempAlert(...));
        }
    }
}

// in EventServiceProvider
protected $listen = [
    DataChangeReceived::class => [TemperatureMonitor::class],
];
```

#### Symfony

`symfony-opcua` wires `EventDispatcher`:

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use PhpOpcua\Client\Event\AlarmActivated;

#[AsEventListener]
class AlarmHandler {
    public function __invoke(AlarmActivated $event): void { /* ... */ }
}
```

#### Plain PHP

Any PSR-14 dispatcher works. `crell/tukio`, `php-di/event-dispatcher`, etc.

### Tuning

- **Publishing interval** comes from each subscription's `revisedPublishingInterval` — set when calling `createSubscription($interval)`.
- The AutoPublisher does NOT busy-loop. Between publishing ticks the React event loop sleeps.
- `SubscriptionKeepAlive` events fire when the publishing interval elapses with no notifications — useful for liveness checks.
- If the daemon's PSR-14 dispatcher throws, the exception is logged and swallowed — listener errors never break the publish cycle.

### Stopping

The AutoPublisher stops automatically when:

- The session is closed (explicit `disconnect()` or inactivity timeout)
- The subscription is deleted (`deleteSubscriptions`)
- The daemon receives SIGTERM / SIGINT — every subscription terminates cleanly during the shutdown sequence

## Auto-connect

### Problem it solves

Eagerly establish sessions at daemon startup — so the first user request hits warm sessions instead of paying the connect handshake. Useful for known endpoints (one PLC per shopfloor, one historian per site).

### Wiring (CLI configuration file)

Define a JSON / PHP file describing connections + subscriptions:

```json
{
  "connections": [
    {
      "endpointUrl": "opc.tcp://plc1.example:4840",
      "security": {
        "policy": "Basic256Sha256",
        "mode": "SignAndEncrypt",
        "clientCertPath": "/etc/opcua/certs/client.pem",
        "clientKeyPath": "/etc/opcua/certs/client.key"
      },
      "identity": {
        "username": "operator",
        "password": "..."
      },
      "subscriptions": [
        {
          "publishingInterval": 500.0,
          "monitoredItems": [
            { "nodeId": "ns=2;s=Plant/Temp", "samplingInterval": 500.0, "queueSize": 10 },
            { "nodeId": "ns=2;s=Plant/Pressure" }
          ]
        }
      ]
    }
  ]
}
```

### Wiring (programmatic)

```php
use PhpOpcua\SessionManager\Daemon\SessionManagerDaemon;
use PhpOpcua\SessionManager\Daemon\SessionConfig;

$autoConnect = [
    new SessionConfig(
        endpointUrl: 'opc.tcp://plc1.example:4840',
        securityPolicy: 'Basic256Sha256',
        securityMode: 'SignAndEncrypt',
        clientCertPath: '/etc/opcua/certs/client.pem',
        clientKeyPath: '/etc/opcua/certs/client.key',
        username: 'operator',
        password: $_ENV['PLC_PASSWORD'],
        subscriptions: [
            ['publishingInterval' => 500.0, 'monitoredItems' => [
                ['nodeId' => 'ns=2;s=Plant/Temp'],
                ['nodeId' => 'ns=2;s=Plant/Pressure'],
            ]],
        ],
    ),
];

$daemon = new SessionManagerDaemon(
    socketPath: '/var/run/opcua/sm.sock',
    clientEventDispatcher: $dispatcher,
    autoPublish: true,
    autoConnect: $autoConnect,
);
$daemon->run();
```

### Behaviour

1. After the IPC listener binds, the daemon iterates `autoConnect` configs.
2. Each opens a session + creates the declared subscriptions.
3. AutoPublisher tracks them immediately — events start firing before any client request arrives.
4. On reconnect failures, the daemon retries with exponential backoff. `ConnectionFailed` PSR-14 event tells your listener what went wrong.

### Framework idioms

In `laravel-opcua` / `symfony-opcua`, auto-connect is declared in the same config file as normal connections:

```yaml
# Symfony
php_opcua_symfony_opcua:
    connections:
        plc1:
            endpoint: '%env(PLC1_ENDPOINT)%'
            auto_connect: true
            subscriptions:
                - publishing_interval: 500
                  monitored_items:
                      - { node_id: 'ns=2;s=Plant/Temp' }
```

```php
// Laravel
return [
    'connections' => [
        'plc1' => [
            'endpoint' => env('PLC1_ENDPOINT'),
            'auto_connect' => true,
            'subscriptions' => [
                ['publishing_interval' => 500, 'monitored_items' => [
                    ['node_id' => 'ns=2;s=Plant/Temp'],
                ]],
            ],
        ],
    ],
];
```

The framework integration translates these into daemon-side `SessionConfig` instances at boot.

## Combining auto-publish with auto-connect

The common production pattern: **declare connections + subscriptions in config; let the daemon manage them; subscribe Laravel/Symfony event listeners to the PSR-14 events**. Your application code never opens a session or calls `publish()` — it just listens.

```
config/opcua.php (declarative) → daemon → AutoPublisher → PSR-14 dispatcher → Laravel listeners
                                                                              ↓
                                                                          your handlers
```

## What auto-publish does NOT do

- It does NOT call `historyRead` / `read` / `write` automatically — those are explicit IPC commands from `ManagedClient`.
- It does NOT survive a daemon restart's subscription state — the daemon reconnects, recreates the subscriptions per auto-connect config, but in-flight notifications that hadn't reached your listener are lost. Plan for at-least-once or at-most-once semantics depending on your downstream.
- It does NOT propagate errors via PSR-14 by default — wire your own listener for `ConnectionFailed`, `SessionClosed`, etc. if you need an error funnel.
