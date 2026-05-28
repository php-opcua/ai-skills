# Enums reference

How OPC UA Enumeration types map to PHP enums, and how the auto-cast works.

## Generator output

For every `<UADataType>` with `IsAbstract=false` and a `<Definition IsOptionSet="false">` containing `<Field>` children, the generator emits a PHP `enum: int`:

```php
namespace PhpOpcua\Nodeset\Robotics\Enums;

/**
 * OPC UA enumeration type: AxisMotionProfileEnumeration.
 *
 * @generated
 */
enum AxisMotionProfileEnumeration: int
{
    case OTHER = 0;
    case ROTARY = 1;
    case ROTARY_ENDLESS = 2;
    case LINEAR = 3;
    case LINEAR_ENDLESS = 4;
}
```

### Naming conventions

- **Class name**: copied verbatim from the spec's DataType name (typically suffixed `Enumeration`).
- **Case names**: TitleCase with underscore separation for multi-word values — `MANUAL_HIGH_SPEED`, not `MANUALHIGHSPEED` or `manualHighSpeed`. Deterministic across runs.
- **Backing type**: always `int` (OPC UA Enumerations are Int32 on the wire).
- **Underlying values**: copied from `<Field Value="N">` exactly.

### File layout

```
src/<Spec>/Enums/<EnumName>.php
```

One file per enum. 309 files total across 51 specs.

## Auto-cast pipeline

When you `loadGeneratedTypes(new <Spec>Registrar())`, each registrar's `getEnumMappings()` returns:

```php
public function getEnumMappings(): array
{
    return [
        'ns=20;i=3000' => AxisMotionProfileEnumeration::class,
        'ns=20;i=3001' => OperationalModeEnumeration::class,
        // …
    ];
}
```

The map keys are the canonical NodeIds of the enum's DataType. The client merges all registrars' maps into one table.

On a `read()`, the client:

1. Determines the node's `DataType` attribute (cached after first read per node)
2. If the `DataType` NodeId is in the enum table → casts the Int32 return value through `<EnumClass>::from($int)`
3. Returns the typed case as the unwrapped value of the `DataValue`

```php
$mode = $client->read('ns=2;s=OperationalMode')->getValue();
// $mode === OperationalModeEnumeration::MANUAL_HIGH_SPEED  (typed case, not int)
```

If the value is outside the defined cases (e.g. server returns a future-spec value not yet in the enum), `from()` would throw — the client catches and falls back to the raw int with a `WARNING` log line.

## Explicit conversion

When you receive an int from an external source (CLI arg, JSON payload, custom protocol):

```php
$case = OperationalModeEnumeration::from($int);          // throws \ValueError on unknown
$case = OperationalModeEnumeration::tryFrom($int);       // returns null on unknown
```

`tryFrom()` is safer for inputs you don't control.

## Iteration

```php
foreach (OperationalModeEnumeration::cases() as $case) {
    echo "{$case->name} = {$case->value}\n";
}
```

`->name` is the PHP case name (e.g. `MANUAL_HIGH_SPEED`). `->value` is the OPC UA integer.

## Comparison

PHP enums support `===` directly:

```php
if ($mode === OperationalModeEnumeration::AUTOMATIC) { /* ... */ }

match ($mode) {
    OperationalModeEnumeration::MANUAL_HIGH_SPEED, OperationalModeEnumeration::MANUAL_LOW_SPEED
        => $this->stopMachine(),
    OperationalModeEnumeration::AUTOMATIC, OperationalModeEnumeration::SEMI_AUTOMATIC
        => $this->resume(),
    default
        => $this->log("Unknown mode: {$mode->name}"),
};
```

## Writing an enum value

Use the case directly — the client's `write()` resolves the underlying int via the registered enum mapping and the DataType:

```php
$client->write('ns=2;s=OperationalMode', OperationalModeEnumeration::AUTOMATIC);
// → serializes Int32(0) on the wire (or whatever the case value is)
```

The reverse mapping is part of the same enum-mapping table; no extra setup.

For explicit control (rare):

```php
use PhpOpcua\Client\Types\BuiltinType;
$client->write('ns=2;s=OperationalMode', OperationalModeEnumeration::AUTOMATIC->value, BuiltinType::Int32);
```

## Where enums live

| Spec | Notable enums |
| --- | --- |
| `Robotics` | `OperationalModeEnumeration`, `AxisMotionProfileEnumeration`, `TaskControlStateEnumeration`, … |
| `MachineTool` | `EquipmentStateMachineEnumeration`, `MaintenanceStateEnumeration`, `ProductionStateMachineEnumeration`, … |
| `Machinery` | `MachineryItemState_StateMachineEnumeration`, `ExecutionState`, `OperationModeStateMachine`, … |
| `DI` | `DeviceHealthEnumeration`, `FetchResultErrorDataType`, … |
| `PackML` | `PackMLBaseStateMachineModeIDEnum`, `PackMLBaseStateMachineStateIDEnum`, … |
| `AutoID` | `RfidPasswordType`, `ScanResultDataType`, … (some are Structures, not enums) |
| `BACnet` | (mostly Structure types; BACnet has fewer pure enums than newer specs) |
| `MachineVision` | `RecipeStateMachineEnumeration`, `ResultStateMachineEnumeration`, … |

Use `find src/<Spec>/Enums/ -name '*.php'` to list a spec's enums locally.

## Tips

- **TitleCase-with-underscores is not arbitrary** — it's the canonical PHP enum case naming convention. Don't expect `lowerCamel`.
- **Re-generation is idempotent** — if a spec rev adds a new case, only that case changes. Existing case values are stable across generations.
- **An enum's NodeId is namespace-prefixed** in the constants (`'ns=20;i=3000'`). At runtime that namespace index may differ — the auto-cast logic remaps via `resolveNodeId()`.
- **Cases marked `@deprecated` by the spec are still generated** as cases. The PHP enum doesn't surface deprecation; that's metadata in the XML you'd need to check separately.
- **Server-defined custom enums** (vendor extensions that aren't in `UA-Nodeset/`) are NOT generated. Use `opcua-cli generate:nodeset <vendor.NodeSet2.xml>` against the vendor's XML.
