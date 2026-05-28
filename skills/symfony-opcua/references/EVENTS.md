# Events

The bundle injects Symfony's `EventDispatcherInterface` into the underlying `Client` / `ManagedClient`. Every event the core dispatches becomes a Symfony event with the same FQCN.

## Event namespace

All event classes live under `PhpOpcua\Client\Event\*`. They are immutable `final readonly` DTOs — public readonly properties, no getters.

## All 56 events

### Connection lifecycle (6)

| Class | When | Key properties |
|---|---|---|
| `ClientConnecting` | Before secure-channel + session creation | `endpointUrl` |
| `ClientConnected` | After session activation | `endpointUrl`, `sessionId` (NodeId) |
| `ClientDisconnecting` | Before close | `endpointUrl` |
| `ClientDisconnected` | After socket close | `endpointUrl` |
| `ClientReconnecting` | When `auto_retry` kicks in | `endpointUrl`, `attempt` (int) |
| `ConnectionFailed` | Connect attempt threw | `endpointUrl`, `exception` |

### Secure channel + session (5)

| Class | When |
|---|---|
| `SecureChannelOpened` | OpenSecureChannelResponse received |
| `SecureChannelClosed` | CloseSecureChannelRequest sent |
| `SessionCreated` | CreateSessionResponse received (before activation) |
| `SessionActivated` | ActivateSessionResponse received |
| `SessionClosed` | CloseSessionResponse received |

### Read / write (5)

| Class | Key properties |
|---|---|
| `NodeValueRead` | `nodeId`, `attributeId`, `dataValue` |
| `NodeValueWritten` | `nodeId`, `attributeId`, `value`, `dataType` (BuiltinType) |
| `NodeValueWriteFailed` | `nodeId`, `value`, `statusCode` |
| `WriteTypeDetecting` | `nodeId` |
| `WriteTypeDetected` | `nodeId`, `dataType` |

### Browse (1)

| Class | Key properties |
|---|---|
| `NodeBrowsed` | `nodeId`, `direction`, `references` (ReferenceDescription[]) |

### Subscriptions (5)

| Class | Key properties |
|---|---|
| `SubscriptionCreated` | `subscriptionId`, `revisedPublishingInterval` |
| `SubscriptionDeleted` | `subscriptionId` |
| `SubscriptionKeepAlive` | `subscriptionId`, `sequenceNumber` |
| `SubscriptionTransferred` | `subscriptionId`, `availableSequenceNumbers` |
| `PublishResponseReceived` | Any PublishResponse, including with data + keep-alive |

### Monitored items (4)

| Class | Key properties |
|---|---|
| `MonitoredItemCreated` | `subscriptionId`, `monitoredItemId`, `nodeId`, `clientHandle` |
| `MonitoredItemModified` | `subscriptionId`, `monitoredItemId`, `revisedSamplingInterval` |
| `MonitoredItemDeleted` | `subscriptionId`, `monitoredItemId` |
| `TriggeringConfigured` | `subscriptionId`, `triggeringItemId`, `addedLinks`, `removedLinks` |

### Notifications (2) — **the ones you most often listen to**

| Class | Key properties |
|---|---|
| `DataChangeReceived` | `subscriptionId`, `clientHandle`, `dataValue` (DataValue: value/timestamp/statusCode) |
| `EventNotificationReceived` | `subscriptionId`, `clientHandle`, `selectFields` (string[]), `eventFields` (mixed[]) |

### Alarms (Part 9) — 9

| Class | When |
|---|---|
| `AlarmEventReceived` | Generic alarm event (raw fields) |
| `AlarmActivated` | Active state went true |
| `AlarmDeactivated` | Active state went false |
| `AlarmAcknowledged` | Operator acknowledged |
| `AlarmConfirmed` | Operator confirmed |
| `AlarmShelved` | Operator shelved (silenced) |
| `AlarmSeverityChanged` | Severity field changed |
| `LimitAlarmExceeded` | A `LimitAlarmType` crossed a limit (`level`: High/Low/HighHigh/LowLow) |
| `OffNormalAlarmTriggered` | An `OffNormalAlarmType` left its normal state |

### History (4)

| Class | When (Part 11 §6.9) |
|---|---|
| `HistoryDataUpdated` | HistoryUpdate `UpdateData` action result Good |
| `HistoryDataDeleted` | HistoryUpdate `DeleteRawModified` or `DeleteAtTime` action Good |
| `HistoryEventUpdated` | HistoryUpdate `UpdateEvent` action Good |
| `HistoryEventDeleted` | HistoryUpdate `DeleteEvent` action Good |

### File transfer (4) — Part 5 §C.2/§C.3

| Class | When |
|---|---|
| `FileOpened` | `fileNodeId`, `fileHandle`, `mode` |
| `FileClosed` | OpenFile / CloseFile Call succeeded |
| `FileBytesRead` | `byteCount` |
| `FileBytesWritten` | `byteCount` |

### Aggregates (1) — Part 13

| Class | When |
|---|---|
| `AggregateComputed` | Client-side aggregate finished — `function` (AggregateFunction), `inputCount`, `outputCount` |

### Discovery (1)

| Class | When |
|---|---|
| `DataTypesDiscovered` | `discoverDataTypes()` finished — `namespaceIndex`, `count` |

### Cache (2)

| Class | When |
|---|---|
| `CacheHit` | Read served from metadata cache |
| `CacheMiss` | Cache miss, going to server |

### Retry (2)

| Class | When |
|---|---|
| `RetryAttempt` | A request is being retried — `attempt`, `delayMs` |
| `RetryExhausted` | Final attempt failed — `attempts`, `exception` |

### Trust store (5)

| Class | When |
|---|---|
| `ServerCertificateAutoAccepted` | TOFU accept on first contact |
| `ServerCertificateManuallyTrusted` | `trustCertificate()` call succeeded |
| `ServerCertificateRejected` | Server cert validation failed (untrusted, expired, hostname mismatch) |
| `ServerCertificateRemoved` | `untrustCertificate()` |
| `ServerCertificateTrusted` | Cert was already in the trust store — no-op |

## Listener registration

### `#[AsEventListener]` (recommended)

```php
namespace App\EventListener;

use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class StoreSensorReading
{
    public function __invoke(DataChangeReceived $event): void
    {
        // ...
    }
}
```

The class is auto-tagged. Method `__invoke` is the default; override with `method:` and `event:` attribute args.

### Multi-listener / explicit method

```php
#[AsEventListener(event: AlarmActivated::class, method: 'onActivated', priority: 100)]
#[AsEventListener(event: AlarmDeactivated::class, method: 'onDeactivated')]
final class AlarmAuditListener
{
    public function onActivated(AlarmActivated $event): void { /* ... */ }
    public function onDeactivated(AlarmDeactivated $event): void { /* ... */ }
}
```

### `services.yaml` tag

```yaml
services:
    App\EventListener\StoreSensorReading:
        tags:
            - { name: kernel.event_listener, event: PhpOpcua\Client\Event\DataChangeReceived }
```

Use this when you can't add attributes (e.g., listener lives in a vendor package).

## Resolving the client_handle back to a node

```yaml
parameters:
    app.opcua_handles:
        1: 'ns=2;s=Temperature'
        2: 'ns=2;s=Pressure'
        10: 'i=2253'
```

```php
public function __construct(
    #[Autowire('%app.opcua_handles%')]
    private array $handles,
) {}

public function __invoke(DataChangeReceived $event): void
{
    $nodeId = $this->handles[$event->clientHandle] ?? null;
    // ...
}
```

For dynamic mappings (multi-tenant), store the map in Doctrine or Redis.

## Heavy listener work → Messenger

Synchronous listeners run inside the daemon's publish loop. Dispatch to Messenger to isolate:

```php
namespace App\Message;

final class SensorReadingReceived
{
    public function __construct(
        public readonly int $clientHandle,
        public readonly mixed $value,
        public readonly ?\DateTimeImmutable $sampledAt,
    ) {}
}
```

```php
namespace App\EventListener;

use App\Message\SensorReadingReceived;
use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Messenger\MessageBusInterface;

#[AsEventListener]
final class DispatchSensorReadingMessage
{
    public function __construct(private MessageBusInterface $bus) {}

    public function __invoke(DataChangeReceived $event): void
    {
        $this->bus->dispatch(new SensorReadingReceived(
            clientHandle: $event->clientHandle,
            value: $event->dataValue->getValue(),
            sampledAt: $event->dataValue->sourceTimestamp,
        ));
    }
}
```

```php
namespace App\MessageHandler;

use App\Message\SensorReadingReceived;
use App\Repository\SensorReadingRepository;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class StoreSensorReadingHandler
{
    public function __construct(private SensorReadingRepository $repo) {}

    public function __invoke(SensorReadingReceived $message): void
    {
        $this->repo->save(/* ... */);
    }
}
```

Route the message to a dedicated transport so a slow PLC doesn't back-pressure the main queue:

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            opcua_data: '%env(MESSENGER_OPCUA_DATA_DSN)%'
        routing:
            App\Message\SensorReadingReceived: opcua_data
```

## Listening across all subscriptions vs filtering

`DataChangeReceived` fires for every notification on every subscription. To filter, check `subscriptionId` inside the listener:

```php
public function __invoke(DataChangeReceived $event): void
{
    if ($event->subscriptionId !== 42) return;
    // ...
}
```

For per-subscription routing, prefer separate listener classes with different priorities + a state-based early return.

## Stopping event propagation

Events are PSR-14 stoppable. The built-in DTOs are plain `final readonly`, so listener priority is the simpler control:

```php
#[AsEventListener(event: DataChangeReceived::class, priority: 100)]
final class HighPriorityListener { /* ... */ }
```

Higher priority runs first. To stop later listeners, type-hint the event with `\Symfony\Contracts\EventDispatcher\Event` and call `stopPropagation()` — but the event must implement that contract; the OPC UA event DTOs do not by default.

## Catching all alarm subtypes

You can listen to the base `AlarmEventReceived` for every alarm subtype:

```php
#[AsEventListener]
final class CatchAllAlarmsListener
{
    public function __invoke(AlarmEventReceived $event): void
    {
        // Fires for AlarmActivated, AlarmDeactivated, AlarmAcknowledged, ...
    }
}
```

All specific alarm events extend `AlarmEventReceived` (verify in `vendor/php-opcua/opcua-client/src/Event/`).

## Memory considerations

Each event holds references to `DataValue` / `Variant` / strings. For >1000 events/sec sustained, prefer Messenger dispatch and avoid keeping closures with captured event references in long-lived services.
