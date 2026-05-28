# Pitfalls reference

Mistakes AI coding assistants frequently make with `opcua-client-nodeset`. Read before generating code.

## 1. Hand-editing files under `src/`

**Wrong**:
```php
// src/Robotics/DataTypes/AxisInformation.php — manually edited
final readonly class AxisInformation implements WireSerializable
{
    public function __construct(
        public string $name,
        public float $minValue,
        public float $maxValue,
        public string $myCustomField,   // ← added by hand
    ) {}
    // …
}
```

Next `composer generate` overwrites your edit silently. Every PR that touches `src/` directly fails CI's `git diff --exit-code` check.

**Right**: edit `generate.php` if generation logic needs to change, OR add a wrapper in your application namespace, OR use `opcua-cli generate:nodeset` against a vendor XML to produce parallel types under your own namespace.

## 2. Registering codecs manually when a registrar exists

**Wrong**:
```php
$client = ClientBuilder::create()->connect('opc.tcp://...');
$client->getRepository()->register(new AxisInformationCodec());
$client->getRepository()->register(new MotionDeviceCodec());
// …repeat for every codec
```

The registrar already does this. Plus you miss the dependency resolution (DI's `EngineeringUnit` codec, Machinery's base types).

**Right**:
```php
$client = ClientBuilder::create()
    ->loadGeneratedTypes(new RoboticsRegistrar())            // walks the dep graph
    ->connect('opc.tcp://...');
```

## 3. Using `only: true` in production

**Wrong**:
```php
$builder->loadGeneratedTypes(new RoboticsRegistrar(only: true));
```

`only: true` skips `dependencyRegistrars()`. The Robotics spec references `DI/EngineeringUnit`, `Machinery/MachineryItemIdentificationType`, etc. Without their codecs registered, those fields decode as raw `ExtensionObject`s.

**Right**: omit `only:` (defaults to `false`):
```php
$builder->loadGeneratedTypes(new RoboticsRegistrar());
```

`only: true` is a debug knob — useful for isolating "does Robotics codec X work without Machinery interfering?". Never in production.

## 4. Hard-coding namespace indices from the spec constants

**Wrong**:
```php
$client->read('ns=20;i=1004');                               // copying from the NodeIds class
// → fails if the server assigns Robotics to a different namespace index
```

The `ns=20` in `RoboticsNodeIds::MOTION_DEVICE_SYSTEM_TYPE = 'ns=20;i=1004'` is the **canonical** index per the OPC Foundation. Your server may assign Robotics to `ns=4` (or any other index, depending on its `NamespaceArray` ordering).

**Right**: use the constant; the typed-read pipeline remaps:
```php
$client->read(RoboticsNodeIds::MOTION_DEVICE_SYSTEM_TYPE);
```

The remap happens inside `Client::read()` — it walks `Server_NamespaceArray` (cached after first read) and rewrites the namespace index to match the server's actual mapping.

For ad-hoc reads where you don't have a constant, use `resolveNodeId()` with a path:
```php
$nodeId = $client->resolveNodeId('/Objects/MyRobot/MotionDevice');
$client->read($nodeId);
```

## 5. Forgetting to handle reads that return raw `ExtensionObject`

When a node's DataType isn't covered by any loaded registrar, `getValue()` returns a raw `ExtensionObject`, not a DTO:

```php
$value = $client->read('ns=2;s=Custom')->getValue();
// $value might be: \PhpOpcua\Client\Types\ExtensionObject (raw)
// OR: \PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation (decoded)
```

**Right** — defensive code:
```php
$value = $client->read('ns=2;s=Custom')->getValue();
if ($value instanceof \PhpOpcua\Client\Types\ExtensionObject) {
    throw new \LogicException(
        "Missing codec for {$value->typeId}. Did you forget loadGeneratedTypes()?"
    );
}
// safe to use $value as the typed DTO
```

Or assert the expected type:
```php
$axis = $client->read('ns=2;s=Axis1')->getValue();
assert($axis instanceof AxisInformation, 'Robotics registrar not loaded');
```

## 6. Cross-spec type imports

When a DTO references a type from another spec (very common — DI types appear everywhere), import from the source namespace:

**Wrong**:
```php
namespace App;

use PhpOpcua\Nodeset\Robotics\DataTypes\EngineeringUnit;   // doesn't exist there
```

`EngineeringUnit` is a DI type, not a Robotics type. Robotics' DTOs import it from `\PhpOpcua\Nodeset\DI\Enums\EngineeringUnit`.

**Right**:
```php
use PhpOpcua\Nodeset\DI\Enums\EngineeringUnit;
```

Check the DTO's `use` statements at the top of the file to find the canonical import.

## 7. Misusing case constants

Enum cases are TitleCase with underscores:

**Wrong**:
```php
OperationalModeEnumeration::manualHighSpeed;          // camelCase — doesn't exist
OperationalModeEnumeration::MANUALHIGHSPEED;          // no underscore — doesn't exist
OperationalModeEnumeration::Manual_High_Speed;        // mixed — doesn't exist
```

**Right**:
```php
OperationalModeEnumeration::MANUAL_HIGH_SPEED;        // canonical case name
```

The case names come from the OPC UA `<Field DisplayName="...">` with deterministic TitleCase conversion: words separated by camelCase boundaries or spaces become uppercase tokens joined by `_`.

## 8. Comparing enum cases via `->value`

**Wrong**:
```php
if ($mode->value === 2) { /* AUTOMATIC, but unreadable */ }
```

Works, but loses every benefit of typed enums (no autocomplete, no refactor safety).

**Right**:
```php
if ($mode === OperationalModeEnumeration::AUTOMATIC) { /* ... */ }
```

For multi-branch logic use `match`:
```php
match ($mode) {
    OperationalModeEnumeration::AUTOMATIC, OperationalModeEnumeration::SEMI_AUTOMATIC => ...,
    OperationalModeEnumeration::MANUAL_HIGH_SPEED => ...,
    default => ...,
};
```

## 9. Writing an enum case as an int

**Wrong**:
```php
$client->write('ns=2;s=Mode', 2);                            // hopes 2 = AUTOMATIC, but magic
```

Brittle: a spec rev could renumber. Loses the type safety.

**Right**:
```php
$client->write('ns=2;s=Mode', OperationalModeEnumeration::AUTOMATIC);
```

The client serializes via the registered enum mapping. If the spec renumbers, your code is correct against the new mapping automatically.

## 10. Loading specs you don't use

```php
$builder->loadGeneratedTypes(
    new RoboticsRegistrar(),
    new MachineToolRegistrar(),
    new BACnetRegistrar(),
    new MTConnectRegistrar(),
    new PackMLRegistrar(),
    new ISA95Registrar(),
    // ...
);
```

Every registrar costs codec-registration time at boot (~milliseconds per codec). Not significant for one or two specs, but at 10+ becomes noticeable. Load only what your app actually decodes.

**Right** — match the registrar list to your domain. A robot integration loads Robotics. A factory monitoring dashboard might load Machinery (for OEE/state) and one or two of the line-specific specs.

## 11. Mixing nodeset versions

If your `composer.json` constrains:
```json
"php-opcua/opcua-client": "^4.4",
"php-opcua/opcua-client-nodeset": "^4.3"
```

Boot may fail: nodeset v4.3 was built against the older `GeneratedTypeRegistrar` interface in core v4.3, which had a slightly different shape (no `getEnumMappings()`). The two move in lock-step.

**Right** — pin to the same minor:
```json
"php-opcua/opcua-client": "^4.4",
"php-opcua/opcua-client-nodeset": "^4.4"
```

## 12. Treating the generator as a library

`generate.php` is a one-shot script, not a library. Don't `require 'generate.php';` from application code. If you need programmatic generation against an arbitrary NodeSet2.xml, use `opcua-cli generate:nodeset` — it wraps the same logic with a stable CLI.

## 13. Assuming all specs have the same maturity

Newer specs (PROFINET v3+, OPC PubSub) have more abstract / union DataTypes and exercise the generator harder. If you hit a `Cannot generate codec for abstract type` error, that's expected — abstract base types don't get codecs, only concrete subtypes do.

Older specs (GPOS, AML) have been stable for years and changes are rare.

## 14. Vendor extensions in `UA-Nodeset/`

**Wrong**: dropping `Siemens.NodeSet2.xml` into `UA-Nodeset/Schema/` and re-generating.

Pollutes the OPC-Foundation-only canonical set. Next time someone updates the `UA-Nodeset` submodule, your Siemens addition gets blown away.

**Right**: use `opcua-cli generate:nodeset` against the vendor XML, producing types under your application's namespace (e.g. `App\OpcUa\Siemens\…`). Commit those in your application repo, not here.

## 15. Forgetting `composer dump-autoload` after generation

Most edits trigger PSR-4 autoload regeneration automatically (composer scripts hook on `post-update`). But if you ONLY ran `composer generate` and a NEW spec directory appeared, you may need:

```bash
composer dump-autoload
```

to make the new classes visible to `\PhpOpcua\Nodeset\…` autoload. The CI workflow does this implicitly because it runs `composer install` before `composer generate`. Locally, after adding a new spec, re-run `composer install` for safety.

## 16. NodeIds constants are strings, not NodeId objects

**Wrong**:
```php
$nodeId = RoboticsNodeIds::MOTION_DEVICE_SYSTEM_TYPE;
echo $nodeId->namespaceIndex;                                // Error: $nodeId is a string
```

**Right** — they're strings; parse if you need an object:
```php
$nodeIdStr = RoboticsNodeIds::MOTION_DEVICE_SYSTEM_TYPE;     // 'ns=20;i=1004'

// Most operations accept the string directly
$client->read($nodeIdStr);

// To get a NodeId object:
$nodeId = NodeId::parse($nodeIdStr);
```

The string-form is preferred for application code (terser, accepted by every client method).
