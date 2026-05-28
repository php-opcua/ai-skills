# DTOs and codecs reference

How OPC UA Structure DataTypes become PHP `readonly class` DTOs, and how their binary codecs decode/encode transparently.

## Generator output for a Structure

For every `<UADataType>` with `IsAbstract=false` and a `<Definition>` containing `<Field>` children typed against other types (not enum cases), the generator emits two files:

**The DTO** — `src/<Spec>/DataTypes/<Name>.php`:

```php
namespace PhpOpcua\Nodeset\Robotics\DataTypes;

use PhpOpcua\Client\Wire\WireSerializable;
use PhpOpcua\Nodeset\DI\Enums\EngineeringUnit;

/**
 * OPC UA Structure DataType: AxisInformation.
 *
 * @generated
 */
final readonly class AxisInformation implements WireSerializable
{
    public function __construct(
        public string $name,
        public float $minValue,
        public float $maxValue,
        public EngineeringUnit $unit,
        public ?string $description = null,
    ) {}

    public static function wireTypeId(): string { return 'PhpOpcua\\Nodeset\\Robotics\\DataTypes\\AxisInformation'; }
    public function jsonSerialize(): array { /* … */ }
    public static function fromWireArray(array $data): static { /* … */ }
}
```

**The binary codec** — `src/<Spec>/Codecs/<Name>Codec.php`:

```php
namespace PhpOpcua\Nodeset\Robotics\Codecs;

use PhpOpcua\Client\Encoding\ExtensionObjectCodec;
use PhpOpcua\Client\Encoding\BinaryEncoder;
use PhpOpcua\Client\Encoding\BinaryDecoder;
use PhpOpcua\Client\Types\NodeId;
use PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation;

/**
 * @generated
 */
final class AxisInformationCodec implements ExtensionObjectCodec
{
    public function typeId(): NodeId { return NodeId::numeric(20, 5020); }   // BinaryEncodingId

    public function encode(mixed $value): string
    {
        assert($value instanceof AxisInformation);
        $encoder = new BinaryEncoder();
        $encoder->writeString($value->name);
        $encoder->writeDouble($value->minValue);
        $encoder->writeDouble($value->maxValue);
        // … field-by-field
        return $encoder->getBuffer();
    }

    public function decode(string $body): AxisInformation
    {
        $decoder = new BinaryDecoder($body);
        return new AxisInformation(
            name: $decoder->readString(),
            minValue: $decoder->readDouble(),
            maxValue: $decoder->readDouble(),
            // … field-by-field
        );
    }
}
```

98 DTO files + 193 codec files total. (More codecs than DTOs because some abstract bases are emitted only as DTOs while their concrete subtypes have codecs.)

## Property conventions

- **All properties are `public readonly`** — no getters, no setters. Access as `$dto->fieldName`.
- **Property names are camelCase** (idiomatic PHP), even when the OPC UA field is PascalCase. The codec handles the mapping.
- **Nullable properties** map to OPC UA "OptionalField" entries in the `<Definition>`. Required fields are non-nullable.
- **Cross-spec references** use the full namespace (`\PhpOpcua\Nodeset\DI\Enums\EngineeringUnit`). The PHP file declares a `use` import at the top.
- **Arrays** are typed as `array` (PHP doesn't have generics) with a PHPDoc `@var SomeType[]` annotation.

## How auto-decode works

When you `loadGeneratedTypes(new <Spec>Registrar())`, every codec is registered into the per-client `ExtensionObjectRepository` keyed by its `typeId()` (the BinaryEncodingId NodeId of the DataType).

On a `read()`:

1. The server returns a `Variant` of `BuiltinType::ExtensionObject` containing a raw `ExtensionObject(typeId, encoding, body)`.
2. `Client` looks up the repository by `typeId`. If found:
   - Calls `$codec->decode($body)` — returns the typed DTO
   - Wraps the DTO inside the `ExtensionObject` as `$extObj->value`
   - Marks `$extObj->isDecoded() === true`
3. `DataValue::getValue()` sees the decoded ExtensionObject and returns `$extObj->value` directly — the typed DTO.

```php
$axis = $client->read('ns=2;s=Axis1')->getValue();
// $axis instanceof \PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation === true
echo "{$axis->name}: {$axis->minValue} → {$axis->maxValue} {$axis->unit->name}\n";
```

If the registrar isn't loaded — or the codec doesn't exist for this specific spec — `getValue()` returns the raw `ExtensionObject` instead. You can detect that:

```php
$value = $client->read('ns=2;s=Axis1')->getValue();
if ($value instanceof \PhpOpcua\Client\Types\ExtensionObject) {
    throw new \LogicException("Missing codec for {$value->typeId}");
}
```

## Writing a DTO

```php
use PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation;
use PhpOpcua\Nodeset\DI\Enums\EngineeringUnit;

$axis = new AxisInformation(
    name: 'X-Axis',
    minValue: 0.0,
    maxValue: 1500.0,
    unit: EngineeringUnit::MILLIMETRE,
);

$client->write('ns=2;s=Axis1', $axis);
// → the registered codec serializes the DTO, the client wraps it in ExtensionObject + Variant
```

The codec lookup goes the other way during write: `Client` sees a value whose class is in the repository, calls `$codec->encode($value)`, builds the `ExtensionObject` envelope with the right `typeId`.

## Nested Structures

A Structure that contains other Structures decodes transitively:

```php
$tool = $client->read('ns=2;s=CuttingTool')->getValue();
// $tool instanceof \PhpOpcua\Nodeset\CuttingTool\DataTypes\Tool === true
// $tool->geometry instanceof \PhpOpcua\Nodeset\CuttingTool\DataTypes\Geometry === true
// $tool->geometry->cuttingEdges is an AxialOffset[] (typed array)
```

As long as every nested Structure's spec's registrar is loaded (typically auto-via dependencies), the entire tree decodes.

## What doesn't get auto-decoded

- **Abstract Structure types** — the spec sometimes defines a base `Structure` with `IsAbstract=true`. No codec is generated for these; only concrete subtypes get codecs. Reading a node that returns an abstract base (rare) gives you the raw `ExtensionObject` with the concrete subtype's `typeId` — load the concrete spec's registrar.
- **OptionSet DataTypes** — these are bitfields, not Structures. They decode as Int32 / Int64 with no special handling. Treat the value as a bitmask:
  ```php
  use PhpOpcua\Nodeset\DI\Enums\DeviceHealthEnumeration;
  $mask = $client->read('ns=2;s=Health')->getValue();
  if ($mask & DeviceHealthEnumeration::FAILURE->value) { /* ... */ }
  ```
  (For OptionSets the generator emits an int-cased enum the same way it does for Enumerations — the bitwise semantics are not encoded in PHP's type system.)
- **Union DataTypes** — present in newer specs (PROFINET, OPC PubSub). The generator emits the union as a DTO with multiple nullable fields; only one is set per instance. Codec handles the discriminator.

## Wire / IPC compatibility

Every DTO implements `\PhpOpcua\Client\Wire\WireSerializable`. That means:

- It can be cached via PSR-16 (encoded/decoded by `WireCacheCodec` — JSON-gated by the wire allowlist)
- It survives the `opcua-session-manager` IPC round-trip — the daemon registers the same codecs, decodes server-side, re-encodes for IPC
- It can be safely passed through any framework's event bus (Laravel, Symfony) without `unserialize()` involvement

`wireTypeId()` returns the fully-qualified class name. The wire registry uses that as the `__t` discriminator.

## Adding fields to a generated DTO

Don't hand-edit the file. Either:

1. **Upstream change** — the OPC Foundation adds the field to the spec. Update `UA-Nodeset/`, re-run `composer generate`, commit.
2. **Vendor extension** — the field exists in a vendor's NodeSet2 but not upstream. Use `opcua-cli generate:nodeset <vendor.NodeSet2.xml>` to produce a one-off DTO/codec in your application's namespace. Don't pollute `PhpOpcua\Nodeset\`.

## Inspecting a codec at runtime

```php
$repo = $client->getRepository();
foreach ($repo->all() as $typeId => $codec) {
    echo $typeId . ' → ' . $codec::class . "\n";
}
```

Useful for debugging "why is this returning a raw ExtensionObject?" — if the `typeId` isn't in the repo, the registrar for that spec isn't loaded.
