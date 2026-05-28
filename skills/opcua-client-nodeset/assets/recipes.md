# Recipes — complete working examples

Copy-pasteable snippets for common companion-spec integrations.

## R1 — Read a typed enum from a Robotics server

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Nodeset\Robotics\RoboticsRegistrar;
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;

$client = ClientBuilder::create()
    ->loadGeneratedTypes(new RoboticsRegistrar())                    // pulls in DI + Machinery
    ->connect('opc.tcp://robot.example:4840');

try {
    $mode = $client->read('ns=2;s=OperationalMode')->getValue();

    match ($mode) {
        OperationalModeEnumeration::AUTOMATIC      => print "OK to run\n",
        OperationalModeEnumeration::MANUAL_HIGH_SPEED,
        OperationalModeEnumeration::MANUAL_LOW_SPEED => print "Hold — operator override\n",
        default                                     => print "Unknown mode: {$mode->name}\n",
    };
} finally {
    $client->disconnect();
}
```

## R2 — Decode a Structure DataType

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Nodeset\Robotics\RoboticsRegistrar;
use PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation;

$client = ClientBuilder::create()
    ->loadGeneratedTypes(new RoboticsRegistrar())
    ->connect('opc.tcp://...');

$axis = $client->read('ns=2;s=Axis1')->getValue();
assert($axis instanceof AxisInformation, 'Robotics registrar not loaded');

echo "{$axis->name}: range [{$axis->minValue}, {$axis->maxValue}] {$axis->unit->name}\n";
if ($axis->description !== null) {
    echo "  description: {$axis->description}\n";
}
```

## R3 — Load multiple unrelated specs

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Nodeset\MachineTool\MachineToolRegistrar;
use PhpOpcua\Nodeset\BACnet\BACnetRegistrar;
use PhpOpcua\Nodeset\AutoID\AutoIDRegistrar;

// A factory monitoring dashboard reading from CNCs, HVAC, and RFID scanners
$client = ClientBuilder::create()
    ->loadGeneratedTypes(
        new MachineToolRegistrar(),                                  // → MachineTool + Machinery + IA + DI
        new BACnetRegistrar(),                                       // → BACnet + DI (DI deduped)
        new AutoIDRegistrar(),                                       // → AutoID + DI (DI deduped)
    )
    ->connect('opc.tcp://factory-aggregator:4840');
```

DI is registered once across all three branches.

## R4 — Iterate every case of an enum

```php
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;

foreach (OperationalModeEnumeration::cases() as $case) {
    echo "{$case->name} = {$case->value}\n";
}
// OTHER = 0
// MANUAL_HIGH_SPEED = 1
// AUTOMATIC = 2
// ...
```

Useful for building dropdowns / option lists in a UI.

## R5 — Write a typed enum

```php
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;

$client->write('ns=2;s=OperationalMode', OperationalModeEnumeration::AUTOMATIC);
// → serializes Int32(2) on the wire, matching the OPC UA EnumValueType definition
```

The client looks up the case's value through the registered enum mapping and the node's `DataType` attribute.

## R6 — Convert from external int (CLI arg, JSON)

```php
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;

$intFromInput = (int) $_POST['mode'];

$case = OperationalModeEnumeration::tryFrom($intFromInput);
if ($case === null) {
    throw new InvalidArgumentException("Unknown mode: $intFromInput");
}

$client->write('ns=2;s=OperationalMode', $case);
```

`tryFrom()` returns null on unknown ints (safer than `from()` which throws).

## R7 — Walk an address space using NodeId constants

```php
use PhpOpcua\Nodeset\Machinery\MachineryRegistrar;
use PhpOpcua\Nodeset\Machinery\MachineryNodeIds;

$client = ClientBuilder::create()
    ->loadGeneratedTypes(new MachineryRegistrar())
    ->connect('opc.tcp://...');

// Browse from the canonical MachineryItem type definition
$refs = $client->browse(MachineryNodeIds::MACHINERY_ITEM_TYPE);

foreach ($refs as $ref) {
    echo "{$ref->displayName}: {$ref->nodeId}\n";
}
```

Constants survive namespace remapping (the typed-read pipeline handles it).

## R8 — Build a DTO and write it

```php
use PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation;
use PhpOpcua\Nodeset\DI\Enums\EngineeringUnit;

$axis = new AxisInformation(
    name: 'X-Axis',
    minValue: 0.0,
    maxValue: 1500.0,
    unit: EngineeringUnit::MILLIMETRE,
    description: 'Carriage X-axis',
);

$client->write('ns=2;s=Axis1', $axis);
// → codec serializes the DTO, client wraps in ExtensionObject + Variant
```

DTOs are `readonly` — construct with named arguments, can't mutate later. To "update" a value, build a new DTO and write it.

## R9 — Using ManagedClient (session manager)

```php
use PhpOpcua\SessionManager\Client\ManagedClient;
use PhpOpcua\Nodeset\PackML\PackMLRegistrar;
use PhpOpcua\Nodeset\PackML\Enums\PackMLBaseStateMachineStateIDEnum;

$client = new ManagedClient();
$client->loadGeneratedTypes(new PackMLRegistrar());      // codecs registered DAEMON-side, decode on the way back
$client->connect('opc.tcp://...');

$state = $client->read('ns=2;s=PackML/State')->getValue();
assert($state instanceof PackMLBaseStateMachineStateIDEnum);
```

`loadGeneratedTypes()` on `ManagedClient` sends the registrar list to the daemon over IPC. The daemon registers the codecs on its internal `Client`; results decode there and travel back through the wire registry as typed objects.

## R10 — Laravel integration

```php
// config/opcua.php
return [
    'connections' => [
        'robot1' => [
            'endpoint' => env('ROBOT_ENDPOINT'),
            'user' => env('ROBOT_USER'),
            'password' => env('ROBOT_PASSWORD'),
            'generated_types' => [
                \PhpOpcua\Nodeset\Robotics\RoboticsRegistrar::class,
            ],
        ],
    ],
];
```

```php
// app code
use PhpOpcua\LaravelOpcua\Facades\Opcua;
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;

$client = Opcua::connect('robot1');                      // registrars loaded from config
$mode = $client->read('ns=2;s=OperationalMode')->getValue();
assert($mode instanceof OperationalModeEnumeration);
```

`laravel-opcua` v4.4+ reads the `generated_types` array from each connection config and calls `loadGeneratedTypes()` automatically.

## R11 — Symfony integration

```yaml
# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    connections:
        machine1:
            endpoint: '%env(MACHINE_ENDPOINT)%'
            generated_types:
                - PhpOpcua\Nodeset\MachineTool\MachineToolRegistrar
                - PhpOpcua\Nodeset\CuttingTool\CuttingToolRegistrar
```

```php
use PhpOpcua\SymfonyOpcua\OpcuaManager;
use PhpOpcua\Nodeset\MachineTool\DataTypes\Tool;

class ToolListController {
    public function __construct(private OpcuaManager $opcua) {}

    public function __invoke(): JsonResponse {
        $client = $this->opcua->connect('machine1');
        $tool = $client->read('ns=2;s=ActiveTool')->getValue();
        assert($tool instanceof Tool);
        return $this->json(['name' => $tool->name, 'lifeRemaining' => $tool->lifeRemaining]);
    }
}
```

## R12 — Generating types from a vendor's NodeSet2.xml

```bash
# Install the CLI (if not already)
composer require php-opcua/opcua-cli

# Generate against the vendor file — output under YOUR application namespace,
# NOT under PhpOpcua\Nodeset
opcua-cli generate:nodeset /path/to/Vendor.NodeSet2.xml \
    --output=src/Generated/OpcUa \
    --namespace='App\OpcUa\Vendor'
```

The generated `App\OpcUa\Vendor\VendorRegistrar` plugs into `loadGeneratedTypes()` alongside the OPC Foundation registrars:

```php
$client = ClientBuilder::create()
    ->loadGeneratedTypes(
        new \PhpOpcua\Nodeset\Machinery\MachineryRegistrar(),        // upstream
        new \App\OpcUa\Vendor\VendorRegistrar(),                     // vendor-specific
    )
    ->connect('opc.tcp://...');
```

## R13 — Inspecting what got registered

```php
$client = ClientBuilder::create()
    ->loadGeneratedTypes(new RoboticsRegistrar())
    ->connect('opc.tcp://...');

// What's in the per-client codec repository?
foreach ($client->getRepository()->all() as $typeId => $codec) {
    echo $typeId . ' → ' . $codec::class . "\n";
}

// What enum mappings does the client know about?
foreach ($client->getEnumMappings() as $nodeIdStr => $enumClass) {
    echo "$nodeIdStr → $enumClass\n";
}
```

Useful for "why isn't this auto-decoding?" debugging.

## R14 — Re-generating after upstream UA-Nodeset update

```bash
# Pull the latest OPC Foundation XML
cd UA-Nodeset
git pull
cd ..

# Re-generate every src/ file from the new XML
composer generate

# Inspect what changed
git status                                                   # new spec directories?
git diff src/ | head -50                                     # changed enum cases / new fields?

# Test against your server (if applicable)
vendor/bin/pest --group=integration

# Commit
git add UA-Nodeset src/
git commit -m "[UPD] Regenerate against UA-Nodeset $(git -C UA-Nodeset rev-parse --short HEAD)"
```

## R15 — Reading a discovery server's namespaces to pick the right registrars

```php
use PhpOpcua\Client\ClientBuilder;

// Connect to a discovery server with NO registrars first
$client = ClientBuilder::create()->connect('opc.tcp://localhost:4840');

$namespaces = $client->read('i=2255')->getValue();
// Index 0 is always 'http://opcfoundation.org/UA/'
// Other indices map to companion specs / vendor namespaces

// Pick registrars whose namespace URI appears
$registrars = [];
foreach ($namespaces as $uri) {
    $registrars = match (true) {
        str_contains($uri, '/UA/Robotics/')   => [...$registrars, new \PhpOpcua\Nodeset\Robotics\RoboticsRegistrar()],
        str_contains($uri, '/UA/Machinery/')  => [...$registrars, new \PhpOpcua\Nodeset\Machinery\MachineryRegistrar()],
        str_contains($uri, '/UA/MachineTool/') => [...$registrars, new \PhpOpcua\Nodeset\MachineTool\MachineToolRegistrar()],
        // … pick what applies
        default => $registrars,
    };
}

$client->disconnect();

// Reconnect with the right registrars loaded
$client = ClientBuilder::create()
    ->loadGeneratedTypes(...$registrars)
    ->connect('opc.tcp://localhost:4840');
```

Pattern: probe first, load relevant registrars, reconnect with typed-read pipeline.
