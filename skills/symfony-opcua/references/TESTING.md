# Testing

Pest PHP. 5 unit + 9 integration test files. Pest's `KernelTestCase` plus container override is the dominant pattern.

## Pest setup

`tests/Pest.php`:

```php
<?php

uses(\Tests\TestCase::class)->in('Unit', 'Integration');
```

Minimal `TestCase` (from `symfony/framework-bundle`'s `KernelTestCase`):

```php
namespace Tests;

use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

abstract class TestCase extends KernelTestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        self::bootKernel();
    }

    protected function tearDown(): void
    {
        $this->ensureKernelShutdown();
        parent::tearDown();
    }
}
```

`tests/bootstrap.php` (or `tests/Kernel.php`) sets up the minimal `MicroKernelTrait` test kernel that registers `PhpOpcuaSymfonyOpcuaBundle` plus the framework bundle.

## Unit testing the OpcuaManager

```php
use PhpOpcua\SymfonyOpcua\OpcuaManager;

it('returns the default connection name from config', function () {
    $config = [
        'default' => 'plc-1',
        'session_manager' => ['enabled' => false],
        'connections' => ['plc-1' => ['endpoint' => 'opc.tcp://x:4840']],
    ];

    $manager = new OpcuaManager($config);

    expect($manager->getDefaultConnection())->toBe('plc-1');
});
```

For tests that build real connections, mock the `ClientBuilder`:

```php
$builder = Mockery::mock(\PhpOpcua\Client\ClientBuilderInterface::class);
$builder->shouldReceive('build')->andReturn($mockClient);

// Inject via OpcuaManager constructor or a subclass override
```

## MockClient pattern

`PhpOpcua\Client\Testing\MockClient` ships in `opcua-client`. It implements `OpcUaClientInterface`.

```php
use PhpOpcua\Client\OpcUaClientInterface;
use PhpOpcua\Client\Testing\MockClient;
use PhpOpcua\Client\Types\DataValue;

it('renders the server state', function () {
    $mock = MockClient::create()
        ->onRead('i=2259', fn() => DataValue::ofInt32(0));

    self::getContainer()->set(OpcUaClientInterface::class, $mock);

    $client = static::createClient();
    $client->request('GET', '/server-state');

    expect($client->getResponse()->getStatusCode())->toBe(200);
});
```

The crucial line is `self::getContainer()->set(OpcUaClientInterface::class, $mock)` — this overrides the autowired service for the rest of the test.

## MockClient with multiple expectations

```php
$mock = MockClient::create()
    ->onRead('ns=2;s=Temp', fn() => DataValue::ofDouble(72.5))
    ->onRead('ns=2;s=Pressure', fn() => DataValue::ofDouble(101.3))
    ->onWrite('ns=2;s=Setpoint', fn($value) => 0);  // 0 = Good StatusCode

self::getContainer()->set(OpcUaClientInterface::class, $mock);

// Run code under test...

expect($mock->callCount('write'))->toBe(1);
expect($mock->lastWriteValue('ns=2;s=Setpoint'))->toBe(42.5);
```

## Available DataValue factories

```php
DataValue::ofBoolean(true);
DataValue::ofInt16(42);
DataValue::ofUInt16(42);
DataValue::ofInt32(42);
DataValue::ofUInt32(42);
DataValue::ofDouble(3.14);
DataValue::ofFloat(3.14);
DataValue::ofString('hello');
DataValue::of(BuiltinType::Int64, 1234567890);
DataValue::bad(0x80000000);
```

## Testing event listeners

```php
use App\EventListener\StoreSensorReading;
use PhpOpcua\Client\Event\DataChangeReceived;
use PhpOpcua\Client\Types\DataValue;

it('stores readings on DataChangeReceived', function () {
    $event = new DataChangeReceived(
        subscriptionId: 1,
        clientHandle: 5,
        dataValue: DataValue::ofDouble(72.5),
    );

    $listener = self::getContainer()->get(StoreSensorReading::class);
    ($listener)($event);

    $reading = $this->repo->findByClientHandle(5);
    expect($reading?->value)->toBe(72.5);
});
```

## Faking events at the dispatcher level

For higher-level assertions, use Symfony's `EventDispatcherCollectorPass` or override `event_dispatcher` with a test double:

```php
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

it('dispatches DataChangeReceived', function () {
    $dispatcher = new EventDispatcher();
    $captured = [];
    $dispatcher->addListener(DataChangeReceived::class, function ($e) use (&$captured) {
        $captured[] = $e;
    });

    self::getContainer()->set(EventDispatcherInterface::class, $dispatcher);

    // Trigger an action that dispatches the event...

    expect($captured)->toHaveCount(1);
});
```

## Integration tests with the Docker test suite

Requires `php-opcua/uanetstandard-test-suite:v1.5.0` containers running on:

| Port | Server |
|---|---|
| 4840 | Plaintext (None) |
| 4843 | SignAndEncrypt (Basic256Sha256) |
| 4848 | ECC NIST |
| 4849 | ECC Brainpool |
| 4851 | Security Key Service |
| 4852 | HTTPS Binary |
| 24842 | open62541 historizing |

Start:
```bash
docker compose -f compose.test-suite.yml up -d
```

Tag tests with `--group=integration` to skip them locally without Docker:

```php
it('connects to plaintext server', function () { ... })->group('integration');
```

```bash
./vendor/bin/pest tests/Unit/                            # always
./vendor/bin/pest tests/Integration/ --group=integration # when Docker is up
```

`phpunit.xml` default-excludes the `integration` group:

```xml
<phpunit>
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>tests/Integration</directory>
        </testsuite>
    </testsuites>
    <groups>
        <exclude><group>integration</group></exclude>
    </groups>
</phpunit>
```

## Sample integration test

```php
use PhpOpcua\Client\OpcUaClientInterface;

it('reads the ServerStatus state attribute', function () {
    // The test kernel's default connection should point at opc.tcp://localhost:4840
    $client = self::getContainer()->get(OpcUaClientInterface::class);

    $state = $client->read('i=2259')->getValue();

    expect($state)->toBeIn([0, 1, 2, 3, 4, 5, 6, 7]);
})->group('integration');
```

## Testing the SessionCommand

```php
use PhpOpcua\SymfonyOpcua\Command\SessionCommand;
use Symfony\Component\Console\Tester\CommandTester;

it('displays the settings table on dry-run', function () {
    $command = self::getContainer()->get(SessionCommand::class);
    $tester = new CommandTester($command);

    // The command starts a long-running daemon; test only the bootstrap
    // by injecting a daemon mock or by sending SIGTERM after a tick.
});
```

In practice, the command is tested via the daemon mock — see `tests/Unit/SessionCommandTest.php` for the canonical pattern (29 tests covering option parsing, settings-table rendering, auto-publish wiring).

## Bundle-level tests (Configuration tree)

`tests/Unit/ConfigurationTest.php` validates the YAML schema by feeding sample configs through `DefinitionConfigurator` and asserting normalized output:

```php
use Symfony\Component\Config\Definition\Processor;
use Symfony\Component\Config\Definition\Builder\TreeBuilder;

it('accepts a minimal config', function () {
    $tree = $this->buildConfigurationTree();  // helper that mimics AbstractBundle::configure()
    $processed = (new Processor())->process($tree, [['default' => 'default']]);

    expect($processed['default'])->toBe('default');
    expect($processed['session_manager']['enabled'])->toBeTrue();
});
```

## CI matrix

```yaml
strategy:
  matrix:
    php: ['8.2', '8.3', '8.4']
    symfony: ['7.3.*', '7.4.*', '8.0.*']
```

For integration: spin up `uanetstandard-test-suite` as a Docker service and gate `--group=integration` behind it.

## Code coverage

Target ≥ 99.5% (matches the rest of the php-opcua ecosystem). Gaps usually live in `OpcuaManager::configureBuilder()` branches (one per security policy / mode permutation). Don't enumerate every permutation — pick one RSA, one ECC, one None, trust the underlying `opcua-client` test coverage.
