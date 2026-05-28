# Integrations

How `symfony-opcua` plays with the rest of the Symfony ecosystem.

## Symfony Messenger

The natural offload for heavy event handling. See `EVENTS.md` for the dispatch pattern.

Recommended routing:

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            opcua_data: '%env(MESSENGER_OPCUA_DATA_DSN)%'
            opcua_alarms: '%env(MESSENGER_OPCUA_ALARMS_DSN)%'
        routing:
            App\Message\SensorReadingReceived: opcua_data
            App\Message\AlarmRaised: opcua_alarms
```

Worker config:

```ini
# /etc/systemd/system/messenger-opcua.service
[Service]
ExecStart=/usr/bin/php /var/www/app/bin/console messenger:consume opcua_data opcua_alarms --time-limit=3600 --memory-limit=128M
Restart=always
```

If your OPC UA Messenger handlers also do OPC UA writes (e.g., setpoint adjustment), they share the same `OpcuaManager` singleton — works fine.

## API Platform

Expose OPC UA reads as REST/GraphQL via a custom data provider:

```php
namespace App\State;

use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Metadata\Operation;
use PhpOpcua\Client\OpcUaClientInterface;
use App\Dto\TagReading;

final class TagReadingProvider implements ProviderInterface
{
    public function __construct(private readonly OpcUaClientInterface $opcua) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): TagReading
    {
        $dv = $this->opcua->read($uriVariables['nodeId']);
        return new TagReading(
            nodeId: $uriVariables['nodeId'],
            value: $dv->getValue(),
            statusCode: $dv->statusCode,
            sourceTimestamp: $dv->sourceTimestamp,
        );
    }
}
```

```php
namespace App\Dto;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use App\State\TagReadingProvider;

#[ApiResource(operations: [new Get(uriTemplate: '/tags/{nodeId}', provider: TagReadingProvider::class)])]
final class TagReading
{
    public function __construct(
        public readonly string $nodeId,
        public readonly mixed $value,
        public readonly int $statusCode,
        public readonly ?\DateTimeImmutable $sourceTimestamp,
    ) {}
}
```

## EasyAdmin

Inject the manager in a `Dashboard` or `CrudController`:

```php
namespace App\Controller\Admin;

use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
use PhpOpcua\SymfonyOpcua\OpcuaManager;

class DashboardController extends AbstractDashboardController
{
    public function __construct(private OpcuaManager $opcuaManager) {}

    public function configureDashboard(): Dashboard
    {
        $state = $this->opcuaManager->connect()->read('i=2259')->getValue();
        return Dashboard::new()->setTitle("Plant — " . match ($state) {
            0 => 'Running', 1 => 'Failed', default => "State $state",
        });
    }
}
```

For an admin "browse the address space" UI, build a custom `Crud` template that calls `browseRecursive('i=85')`.

## Mercure (server-sent events)

Stream OPC UA data changes to browsers in real time:

```php
namespace App\EventListener;

use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\Update;

#[AsEventListener]
final class PublishToMercure
{
    public function __construct(private HubInterface $hub) {}

    public function __invoke(DataChangeReceived $event): void
    {
        $this->hub->publish(new Update(
            topics: ['opcua://tag/' . $event->clientHandle],
            data: json_encode([
                'value' => $event->dataValue->getValue(),
                'statusCode' => $event->dataValue->statusCode,
                'sourceTimestamp' => $event->dataValue->sourceTimestamp?->format(\DateTimeInterface::ATOM),
            ]),
        ));
    }
}
```

Browser:
```js
const es = new EventSource('/.well-known/mercure?topic=opcua://tag/1');
es.onmessage = e => console.log(JSON.parse(e.data));
```

## Doctrine

Persist event payloads via Messenger handlers (preferred — keeps the daemon publish loop free) or directly in a listener (only for low-rate events like alarms):

```php
namespace App\MessageHandler;

use App\Entity\SensorReading;
use App\Message\SensorReadingReceived;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class StoreSensorReadingHandler
{
    public function __construct(private EntityManagerInterface $em) {}

    public function __invoke(SensorReadingReceived $msg): void
    {
        $this->em->persist(new SensorReading(
            clientHandle: $msg->clientHandle,
            value: (string) $msg->value,
            sampledAt: $msg->sampledAt,
        ));
        $this->em->flush();
    }
}
```

For >100 readings/sec, batch with `EntityManager::clear()` every N persists, or use raw `Connection::executeStatement()` for bulk inserts.

## Monolog channels

A dedicated `opcua` channel keeps the noise out of the main log:

```yaml
# config/packages/monolog.yaml
monolog:
    channels: ['opcua']
    handlers:
        opcua_file:
            type: rotating_file
            path: '%kernel.logs_dir%/opcua.log'
            level: info
            channels: ['opcua']

# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    session_manager:
        log_channel: opcua
    connections:
        default:
            log_channel: opcua
```

The bundle's `LoggerResolverFactory` maps `log_channel: opcua` → `monolog.logger.opcua` at compile time.

## Symfony Cache pools

A dedicated cache pool for OPC UA metadata avoids evicting other app data:

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            opcua.cache:
                adapter: cache.adapter.filesystem
                default_lifetime: 3600

# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    session_manager:
        cache_pool: opcua.cache
```

The bundle wraps `opcua.cache` in `Psr16Cache`.

## FrankenPHP / Octane-style workers

`OpcuaManager` is a public singleton — long-lived across requests in worker mode. Two patterns:

### A — daemon enabled (recommended)

Singleton holds `ManagedClient`. IPC carries session context, no per-request state leaks.

### B — daemon disabled

Singleton holds a direct `Client` shared across requests in the same worker. Use the `kernel.terminate` event to reset:

```php
namespace App\EventSubscriber;

use PhpOpcua\SymfonyOpcua\OpcuaManager;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ResetOpcuaConnectionsSubscriber implements EventSubscriberInterface
{
    public function __construct(private OpcuaManager $opcuaManager) {}

    public function onTerminate(TerminateEvent $event): void
    {
        $this->opcuaManager->disconnectAll();
    }

    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::TERMINATE => 'onTerminate'];
    }
}
```

But this defeats much of the worker-mode benefit. Prefer the daemon.

## Symfony Scheduler

```php
namespace App\Scheduler;

use App\Message\OpcuaProbe;
use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\RecurringMessage;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;

#[AsSchedule]
final class OpcuaProbeSchedule implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return (new Schedule())
            ->add(RecurringMessage::every('60 seconds', new OpcuaProbe()));
    }
}
```

```php
namespace App\MessageHandler;

use App\Message\OpcuaProbe;
use PhpOpcua\Client\OpcUaClientInterface;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class OpcuaProbeHandler
{
    public function __construct(
        private OpcUaClientInterface $opcua,
        private LoggerInterface $logger,
    ) {}

    public function __invoke(OpcuaProbe $msg): void
    {
        try {
            $state = $this->opcua->read('i=2259')->getValue();
            if ($state !== 0) {
                $this->logger->warning('OPC UA server not running', ['state' => $state]);
            }
        } catch (\Throwable $e) {
            $this->logger->error('OPC UA probe failed', ['exception' => $e]);
        }
    }
}
```

## Symfony UX / Turbo

Pair with Mercure for real-time UX:

```twig
{# templates/dashboard.html.twig #}
<div {{ stimulus_controller('opcua-tag', {
    topic: 'opcua://tag/1',
}) }}>
    <span data-opcua-tag-target="value">{{ initialValue }}</span>
</div>
```

```js
// assets/controllers/opcua-tag_controller.js
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
    static targets = ['value'];
    static values = { topic: String };

    connect() {
        const url = new URL('/.well-known/mercure', window.location.origin);
        url.searchParams.append('topic', this.topicValue);
        this.es = new EventSource(url);
        this.es.onmessage = e => {
            const data = JSON.parse(e.data);
            this.valueTarget.textContent = data.value;
        };
    }

    disconnect() { this.es?.close(); }
}
```

## Notifier (alarm-driven notifications)

```php
namespace App\EventListener;

use PhpOpcua\Client\Event\AlarmActivated;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\NotifierInterface;
use Symfony\Component\Notifier\Recipient\Recipient;

#[AsEventListener]
final class NotifyOnAlarm
{
    public function __construct(private NotifierInterface $notifier) {}

    public function __invoke(AlarmActivated $event): void
    {
        $this->notifier->send(
            (new Notification(
                subject: "Alarm: {$event->sourceName}",
                channels: ['chat/slack', 'sms'],
            ))->content((string) $event->message),
            new Recipient(email: 'oncall@plant.example', phone: '+12025551234'),
        );
    }
}
```

## Sentry / observability

OPC UA events are dispatched through Symfony's event dispatcher → Sentry's Symfony bundle captures them if you opt in:

```yaml
# config/packages/sentry.yaml
sentry:
    options:
        before_send: ['App\Sentry\OpcuaEventFilter', 'filter']
```

Or send errors explicitly from listeners that catch `ConnectionFailed`, `RetryExhausted`, etc.

## Multi-tenant (per-tenant connection)

```php
namespace App\Controller;

use App\Repository\TenantRepository;
use PhpOpcua\SymfonyOpcua\OpcuaManager;
use Symfony\Component\Routing\Attribute\Route;

final class TenantPlantController
{
    public function __construct(
        private OpcuaManager $opcuaManager,
        private TenantRepository $tenants,
    ) {}

    #[Route('/tenants/{id}/state')]
    public function state(string $id): array
    {
        $tenant = $this->tenants->find($id);
        $client = $this->opcuaManager->connectTo(
            $tenant->plcEndpoint,
            [
                'security_policy' => 'Basic256Sha256',
                'security_mode' => 'SignAndEncrypt',
                'username' => $tenant->plcUser,
                'password' => $tenant->getDecryptedPlcPass(),
            ],
            as: "tenant:$id",
        );
        return ['state' => $client->read('i=2259')->getValue()];
    }
}
```

The `as:` key scopes the connection per tenant. Across requests under the daemon, sessions are reused.
