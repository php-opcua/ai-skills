# Recipes — copy-pasteable Symfony + OPC UA snippets

Each recipe is a complete end-to-end implementation.

## R1 — Read a single tag from a controller

```php
namespace App\Controller;

use PhpOpcua\Client\OpcUaClientInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

final class ServerStateController extends AbstractController
{
    public function __construct(private readonly OpcUaClientInterface $opcua) {}

    #[Route('/server-state', methods: ['GET'])]
    public function __invoke(): JsonResponse
    {
        $state = $this->opcua->read('i=2259')->getValue();
        return $this->json(['state' => $state, 'running' => $state === 0]);
    }
}
```

## R2 — Read multiple tags in one round-trip

```php
$results = $this->opcua->readMulti()
    ->node('ns=2;s=Temperature')->value()
    ->node('ns=2;s=Pressure')->value()
    ->node('ns=2;s=FlowRate')->value()
    ->node('i=2259')->value()
    ->execute();

return $this->json([
    'temperature' => $results[0]->getValue(),
    'pressure'    => $results[1]->getValue(),
    'flow_rate'   => $results[2]->getValue(),
    'state'       => $results[3]->getValue(),
]);
```

## R3 — Write a setpoint with auto-detected type

```php
use PhpOpcua\Client\Types\StatusCode;

#[Route('/setpoint', methods: ['PUT'])]
public function setSetpoint(Request $request): JsonResponse
{
    $value = $request->toArray()['value'];
    $status = $this->opcua->write('ns=2;s=Setpoint', $value);

    if (! StatusCode::isGood($status)) {
        throw $this->createNotFoundException('Write failed: ' . StatusCode::getName($status));
    }
    return $this->json(['ok' => true]);
}
```

## R4 — Connect to a runtime-discovered endpoint

```php
namespace App\Controller;

use App\Repository\TenantRepository;
use PhpOpcua\SymfonyOpcua\OpcuaManager;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Routing\Attribute\Route;

final class TenantPlantController extends AbstractController
{
    public function __construct(
        private readonly OpcuaManager $opcuaManager,
        private readonly TenantRepository $tenants,
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
        return $this->json(['state' => $client->read('i=2259')->getValue()]);
    }
}
```

## R5 — Browse the Objects folder recursively

```php
#[Route('/address-space')]
public function addressSpace(): JsonResponse
{
    $tree = $this->opcua->browseRecursive('i=85', maxDepth: 3);

    return $this->json(array_map(fn($n) => [
        'browseName' => (string) $n->reference->browseName,
        'nodeId' => (string) $n->reference->nodeId,
        'children' => count($n->children),
    ], $tree));
}
```

## R6 — Auto-publish subscription → Messenger → DB

```yaml
# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    session_manager:
        auto_publish: true
        log_channel: opcua
    connections:
        plc-1:
            endpoint: '%env(PLC1_ENDPOINT)%'
            security_policy: Basic256Sha256
            security_mode: SignAndEncrypt
            username: '%env(PLC1_USER)%'
            password: '%env(PLC1_PASS)%'
            auto_connect: true
            subscriptions:
                - publishing_interval: 500.0
                  monitored_items:
                      - { node_id: 'ns=2;s=Temperature', client_handle: 1, sampling_interval: 250.0 }
                      - { node_id: 'ns=2;s=Pressure', client_handle: 2, sampling_interval: 250.0 }
```

```php
// src/EventListener/DispatchReading.php
namespace App\EventListener;

use App\Message\SensorReadingReceived;
use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Messenger\MessageBusInterface;

#[AsEventListener]
final class DispatchReading
{
    public function __construct(
        private MessageBusInterface $bus,
        #[Autowire('%app.opcua_handles%')]
        private array $handles,
    ) {}

    public function __invoke(DataChangeReceived $event): void
    {
        $this->bus->dispatch(new SensorReadingReceived(
            nodeId: $this->handles[$event->clientHandle] ?? 'unknown',
            value: $event->dataValue->getValue(),
            sampledAt: $event->dataValue->sourceTimestamp ?? new \DateTimeImmutable(),
            quality: $event->dataValue->statusCode,
        ));
    }
}
```

```php
// src/MessageHandler/StoreReadingHandler.php
namespace App\MessageHandler;

use App\Entity\SensorReading;
use App\Message\SensorReadingReceived;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class StoreReadingHandler
{
    public function __construct(private EntityManagerInterface $em) {}

    public function __invoke(SensorReadingReceived $msg): void
    {
        $this->em->persist(new SensorReading(
            nodeId: $msg->nodeId,
            value: (string) $msg->value,
            sampledAt: $msg->sampledAt,
            quality: $msg->quality,
        ));
        $this->em->flush();
    }
}
```

```yaml
# config/services.yaml
parameters:
    app.opcua_handles:
        1: 'ns=2;s=Temperature'
        2: 'ns=2;s=Pressure'
```

## R7 — Mercure broadcast of tag changes

```php
namespace App\EventListener;

use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Mercure\HubInterface;
use Symfony\Component\Mercure\Update;

#[AsEventListener]
final class PublishToMercure
{
    public function __construct(
        private HubInterface $hub,
        #[Autowire('%app.opcua_handles%')]
        private array $handles,
    ) {}

    public function __invoke(DataChangeReceived $event): void
    {
        $nodeId = $this->handles[$event->clientHandle] ?? null;
        if (!$nodeId) return;

        $this->hub->publish(new Update(
            topics: ["opcua://tag/$nodeId"],
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
const es = new EventSource('/.well-known/mercure?topic=opcua://tag/ns=2;s=Temperature');
es.onmessage = e => console.log(JSON.parse(e.data));
```

## R8 — Scheduled probe via Symfony Scheduler

```php
// src/Scheduler/OpcuaProbeSchedule.php
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
// src/MessageHandler/OpcuaProbeHandler.php
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
        private LoggerInterface $opcuaLogger,  // monolog.logger.opcua
    ) {}

    public function __invoke(OpcuaProbe $msg): void
    {
        try {
            $state = $this->opcua->read('i=2259', refresh: true)->getValue();
            if ($state !== 0) {
                $this->opcuaLogger->warning('Server state != Running', ['state' => $state]);
            }
        } catch (\Throwable $e) {
            $this->opcuaLogger->error('OPC UA probe failed', ['exception' => $e]);
        }
    }
}
```

## R9 — EasyAdmin dashboard with live state

```php
namespace App\Controller\Admin;

use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
use PhpOpcua\SymfonyOpcua\OpcuaManager;

class DashboardController extends AbstractDashboardController
{
    public function __construct(private OpcuaManager $opcuaManager) {}

    public function configureDashboard(): Dashboard
    {
        $state = $this->opcuaManager->connect()->read('i=2259')->getValue();
        $label = match ($state) {
            0 => 'Running', 1 => 'Failed', 2 => 'NoConfiguration',
            3 => 'Suspended', 4 => 'Shutdown', 5 => 'Test',
            6 => 'CommunicationFault', 7 => 'Unknown',
            default => "State $state",
        };
        return Dashboard::new()->setTitle("Plant — $label");
    }
}
```

## R10 — API Platform tag-reading endpoint

```php
namespace App\Dto;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use App\State\TagReadingProvider;

#[ApiResource(operations: [
    new Get(uriTemplate: '/tags/{nodeId}', provider: TagReadingProvider::class),
])]
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

```php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use App\Dto\TagReading;
use PhpOpcua\Client\OpcUaClientInterface;

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

## R11 — History read with date range

```php
use Symfony\Component\HttpFoundation\Request;

#[Route('/history/{nodeId}', requirements: ['nodeId' => '.+'])]
public function history(Request $request, string $nodeId): JsonResponse
{
    $start = new \DateTimeImmutable($request->query->get('from', '-1 hour'));
    $end   = new \DateTimeImmutable($request->query->get('to', 'now'));

    $values = $this->opcuaManager->connection('historian')->historyReadRaw($nodeId, $start, $end);

    return $this->json(array_map(fn($dv) => [
        't' => $dv->sourceTimestamp?->format(\DateTimeInterface::ATOM),
        'v' => $dv->getValue(),
        'q' => $dv->statusCode,
    ], $values));
}
```

## R12 — Method call with input arguments

```php
use PhpOpcua\Client\Types\BuiltinType;
use PhpOpcua\Client\Types\StatusCode;
use PhpOpcua\Client\Types\Variant;

#[Route('/motor/start')]
public function startMotor(Request $request): JsonResponse
{
    $rpm = $request->toArray()['rpm'];

    $result = $this->opcua->call(
        objectId: 'ns=2;s=MotorController',
        methodId: 'ns=2;s=MotorController.Start',
        inputArguments: [new Variant(BuiltinType::Int32, $rpm)],
    );

    if (! StatusCode::isGood($result->statusCode)) {
        throw new \RuntimeException("Start failed: " . StatusCode::getName($result->statusCode));
    }
    return $this->json(['outputs' => $result->outputArguments]);
}
```

## R13 — HistoryUpdate (insert backfill data) — v4.4

```php
use PhpOpcua\Client\Types\BuiltinType;
use PhpOpcua\Client\Types\DataValue;
use PhpOpcua\Client\Types\Variant;

#[Route('/backfill', methods: ['POST'])]
public function backfill(Request $request): JsonResponse
{
    $readings = $request->toArray()['readings']; // [{value, ts}, ...]

    $values = array_map(fn($r) => new DataValue(
        value: new Variant(BuiltinType::Double, (float) $r['value']),
        statusCode: 0,
        sourceTimestamp: new \DateTimeImmutable($r['ts']),
    ), $readings);

    $statuses = $this->opcuaManager->connection('historian')->historyInsertData(
        'ns=2;s=Backfill', $values
    );

    return $this->json(['statuses' => $statuses]);
}
```

## R14 — File transfer (download a FileType node) — v4.4

```php
use PhpOpcua\Client\Module\FileTransfer\OpenFileMode;
use Symfony\Component\HttpFoundation\Response;

#[Route('/reports/{nodeId}', requirements: ['nodeId' => '.+'])]
public function download(string $nodeId): Response
{
    $handle = $this->opcua->openFile($nodeId, OpenFileMode::Read);

    $buffer = '';
    while (true) {
        $bytes = $this->opcua->readFile($nodeId, $handle, 65536);
        if ($bytes === '') break;
        $buffer .= $bytes;
    }
    $this->opcua->closeFile($nodeId, $handle);

    return new Response($buffer, 200, ['Content-Type' => 'application/pdf']);
}
```

## R15 — TOFU bootstrap then strict trust

```bash
# Step 1 — bootstrap: dev .env.local
OPCUA_AUTO_ACCEPT=true
```

```yaml
# config/packages/php_opcua_symfony_opcua.yaml
connections:
    default:
        trust_store_path: '%kernel.project_dir%/var/opcua-trust-store'
        auto_accept: '%env(bool:OPCUA_AUTO_ACCEPT)%'
        trust_policy: fingerprint
```

```bash
# Step 2 — first connection auto-trusts
php bin/console debug:container php-opcua  # or trigger a real request

# Step 3 — verify
ls var/opcua-trust-store/trusted/

# Step 4 — flip to strict in prod
# .env.prod or environment variable
OPCUA_AUTO_ACCEPT=false
```

Any cert change after this point throws `UntrustedCertificateException`.

## R16 — Pest test with MockClient

```php
use PhpOpcua\Client\OpcUaClientInterface;
use PhpOpcua\Client\Testing\MockClient;
use PhpOpcua\Client\Types\DataValue;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

it('renders the server state from the controller', function () {
    $client = static::createClient();

    $mock = MockClient::create()
        ->onRead('i=2259', fn() => DataValue::ofInt32(0));
    self::getContainer()->set(OpcUaClientInterface::class, $mock);

    $client->request('GET', '/server-state');

    expect($client->getResponse()->getStatusCode())->toBe(200);
    expect(json_decode($client->getResponse()->getContent(), true))
        ->toBe(['state' => 0, 'running' => true]);
});
```

## R17 — Notifier-based alarm routing

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

## R18 — Healthcheck endpoint

```php
namespace App\Controller;

use PhpOpcua\Client\OpcUaClientInterface;
use PhpOpcua\SymfonyOpcua\OpcuaManager;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

final class HealthController extends AbstractController
{
    public function __construct(
        private OpcUaClientInterface $opcua,
        private OpcuaManager $opcuaManager,
    ) {}

    #[Route('/healthz')]
    public function __invoke(): JsonResponse
    {
        try {
            $state = $this->opcua->read('i=2259', refresh: true)->getValue();
            return new JsonResponse([
                'opcua' => $state === 0 ? 'ok' : "state=$state",
                'session_manager' => $this->opcuaManager->isSessionManagerRunning() ? 'up' : 'down',
            ], $state === 0 ? 200 : 503);
        } catch (\Throwable $e) {
            return new JsonResponse(['opcua' => 'error', 'error' => $e::class], 503);
        }
    }
}
```

## R19 — Custom console command that reads multiple tags

```php
namespace App\Command;

use PhpOpcua\Client\OpcUaClientInterface;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Helper\Table;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand('app:opcua:snapshot', 'Print a snapshot of key tags')]
final class SnapshotCommand extends Command
{
    public function __construct(private OpcUaClientInterface $opcua) { parent::__construct(); }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $results = $this->opcua->readMulti()
            ->node('i=2259')->value()
            ->node('ns=2;s=Temperature')->value()
            ->node('ns=2;s=Pressure')->value()
            ->execute();

        (new Table($output))
            ->setHeaders(['Tag', 'Value', 'StatusCode'])
            ->setRows([
                ['ServerStatus.State', $results[0]->getValue(), $results[0]->statusCode],
                ['Temperature',        $results[1]->getValue(), $results[1]->statusCode],
                ['Pressure',           $results[2]->getValue(), $results[2]->statusCode],
            ])
            ->render();

        return Command::SUCCESS;
    }
}
```

## R20 — Trust-store admin command

```php
namespace App\Command;

use PhpOpcua\Client\OpcUaClientInterface;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand('app:opcua:trust', 'Trust a server cert by file path')]
final class TrustCertCommand extends Command
{
    public function __construct(private OpcUaClientInterface $opcua) { parent::__construct(); }

    protected function configure(): void
    {
        $this->addArgument('path', InputArgument::REQUIRED, 'Path to server cert (DER)');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $der = file_get_contents($input->getArgument('path'));
        $this->opcua->trustCertificate($der);
        $output->writeln('<info>Trusted</info>');
        return Command::SUCCESS;
    }
}
```
