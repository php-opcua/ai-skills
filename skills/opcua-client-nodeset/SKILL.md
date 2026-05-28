---
name: opcua-client-nodeset
description: Drop pre-generated PHP types for 51 OPC Foundation companion specifications (Robotics, MachineTool, DI, Machinery, BACnet, MTConnect, AutoID, ISA-95, PackML, PROFINET, …) into a php-opcua/opcua-client application. One ClientBuilder::loadGeneratedTypes(new RoboticsRegistrar()) call and every read on a structured node returns a typed PHP enum or readonly DTO instead of a raw ExtensionObject. Use this skill whenever an OPC UA task involves a companion specification — typed enums, Structure DataType decoding, well-known NodeId constants, dependency-resolved codec registration, or re-generating PHP from NodeSet2.xml.
license: MIT
compatibility: Requires PHP >= 8.2 and php-opcua/opcua-client v4.4+. Pure PHP — no C extensions. The bundled UA-Nodeset/ XML sources are MIT-licensed by the OPC Foundation.
metadata:
  package: php-opcua/opcua-client-nodeset
  version: v4.4.0
  ecosystem: php-opcua
  generated_files: 807
  specifications: 51
---

# php-opcua/opcua-client-nodeset — v4.4.0 skill

Pre-generated PHP types for 51 OPC Foundation companion specifications. A read-only library — every file in `src/` is the deterministic output of `composer generate` against `UA-Nodeset/`. Plug it into `opcua-client` with one builder call and structured OPC UA values come back as typed PHP enums and `readonly` DTOs.

## When to use this skill

Activate when any of these apply:

- The user mentions a **companion specification** by name (Robotics, MachineTool, Machinery, DI, BACnet, MTConnect, ISA-95, PackML, AutoID, PROFINET, MachineVision, LADS, …)
- They want **typed enums** back from a Read instead of raw integers (`OperationalModeEnumeration::MANUAL_HIGH_SPEED` vs `int(2)`)
- They're reading a **Structure DataType** (custom OPC UA struct) and want a PHP DTO, not a raw `ExtensionObject`
- They reference **well-known NodeIds** from a spec (`MachineryNodeIds::MACHINE_IDENTIFICATION_TYPE`)
- They ask about **regenerating** from a NodeSet2.xml (their own vendor's, or after an upstream `UA-Nodeset` update)
- They use `ClientBuilder::loadGeneratedTypes(...)` or `GeneratedTypeRegistrar`

Do NOT activate for: generic OPC UA tasks not touching companion specs — use the `opcua-client` skill instead.

## The 60-second mental model

```
                  UA-Nodeset/Schema/*.NodeSet2.xml          (OPC Foundation XML — vendored)
                              │
                              ▼   composer generate
                  src/<Spec>/Enums/*.php            ← PHP enums
                  src/<Spec>/DataTypes/*.php        ← readonly DTOs
                  src/<Spec>/Codecs/*.php           ← ExtensionObject codecs
                  src/<Spec>/<Spec>NodeIds.php      ← well-known NodeId constants
                  src/<Spec>/<Spec>Registrar.php    ← implements GeneratedTypeRegistrar
                              │
                              ▼   composer require php-opcua/opcua-client-nodeset
                  Application code:
                      $builder->loadGeneratedTypes(new <Spec>Registrar())
                              │
                              ▼
                  opcua-client walks dependencyRegistrars() transitively
                  → registers every codec into ExtensionObjectRepository
                  → merges enum mappings into the typed-read pipeline
                              │
                              ▼
                  $client->read('ns=2;s=Mode')->getValue()
                  // → OperationalModeEnumeration::AUTOMATIC  (typed!)
```

Three things to know:

1. **Read-only library.** All 807 PHP files are generator output. Don't hand-edit them — they'll be overwritten next regeneration. Hand-edit the generator (`generate.php`) if behavior needs to change.
2. **Registrars walk dependencies themselves.** `new RoboticsRegistrar()` auto-loads DI + Machinery. You almost never need to list multiple registrars unless they're independent branches (e.g. Robotics + BACnet — both depend on DI but on nothing in common otherwise).
3. **Typed reads need both**: the codec (for Structure decode) AND the enum mapping (for Enumeration auto-cast). Both come from the same registrar. Don't register codecs manually if a registrar exists for the spec.

## Quick start (90% of use cases fit this shape)

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Nodeset\Robotics\RoboticsRegistrar;
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;
use PhpOpcua\Nodeset\Robotics\RoboticsNodeIds;

$client = ClientBuilder::create()
    ->loadGeneratedTypes(new RoboticsRegistrar())   // pulls in DI + Machinery transitively
    ->connect('opc.tcp://plc.example:4840');

// Enum auto-cast (Int32 → typed enum case via getEnumMappings())
$mode = $client->read('ns=2;s=OperationalMode')->getValue();
assert($mode === OperationalModeEnumeration::MANUAL_HIGH_SPEED);

// Structure decode (ExtensionObject → typed DTO via registered codec)
$axis = $client->read('ns=2;s=Axis1')->getValue();
echo "Axis range: [{$axis->minValue}, {$axis->maxValue}] {$axis->unit->name}\n";

// Well-known NodeId from the spec
$systemType = $client->read(RoboticsNodeIds::MOTION_DEVICE_SYSTEM_TYPE);
```

## When to load deeper references

| If the task involves… | Read |
| --- | --- |
| Picking the right registrar / understanding the dependency graph (DI vs Machinery vs Robotics) | [`references/REGISTRARS.md`](references/REGISTRARS.md) |
| Working with the generated enums (auto-cast, lookup, all-cases iteration) | [`references/ENUMS.md`](references/ENUMS.md) |
| Working with Structure DTOs and ExtensionObject codecs | [`references/DTOS.md`](references/DTOS.md) |
| Re-generating from XML (composer generate, what `generate.php` does, adding a new spec) | [`references/REGENERATION.md`](references/REGENERATION.md) |
| The 51 spec catalogue — what each spec is for, primary use cases | [`references/CATALOGUE.md`](references/CATALOGUE.md) |
| Debugging unexpected behaviour or common mistakes | [`references/PITFALLS.md`](references/PITFALLS.md) |
| Complete worked examples (Laravel, Symfony, ManagedClient, multi-spec, vendor NodeSet2) | [`assets/recipes.md`](assets/recipes.md) |

## Core API surface (must-know)

Three types matter; nothing else needs to leak into application code.

### `<Spec>Registrar`

Every companion spec has a registrar at `PhpOpcua\Nodeset\<Spec>\<Spec>Registrar`. They all share the same shape:

```php
final class RoboticsRegistrar implements \PhpOpcua\Client\Repository\GeneratedTypeRegistrar
{
    public function __construct(public bool $only = false) {}

    public function registerCodecs(ExtensionObjectRepository $repository): void;
    public function dependencyRegistrars(): array;                          // @return self[]
    public function getEnumMappings(): array;                               // [nodeIdStr => enum-class]
}
```

- `$only = false` (default): `loadGeneratedTypes()` walks `dependencyRegistrars()` transitively.
- `$only = true`: skips dependency resolution. Debug-only — production should always let the graph resolve.

### `<Spec>NodeIds` constants

```php
final class RoboticsNodeIds
{
    public const string MOTION_DEVICE_SYSTEM_TYPE = 'ns=20;i=1004';
    public const string MOTION_DEVICE_TYPE        = 'ns=20;i=1002';
    // …per spec — usually 5–50 well-known IDs
}
```

The `ns=N` here is the **canonical** namespace index per the OPC Foundation. At runtime, your server's actual namespace index may differ — `Client::resolveNodeId()` and the typed-read pipeline auto-remap. Use the constants for readability; the runtime resolution is automatic.

### Generated enums and DTOs

Reachable but rarely written explicitly — the typed-read pipeline gives them to you naturally:

```php
use PhpOpcua\Nodeset\Robotics\Enums\OperationalModeEnumeration;
use PhpOpcua\Nodeset\Robotics\DataTypes\AxisInformation;

OperationalModeEnumeration::from(2);                                       // explicit int → case
foreach (OperationalModeEnumeration::cases() as $case) { /* … */ }

$dto = new AxisInformation(name: 'X', minValue: 0.0, maxValue: 100.0, unit: …);
```

## v4.4.0 alignment

Lock-step with `php-opcua/opcua-client` v4.4.0:

- Uses `ClientBuilder::loadGeneratedTypes(GeneratedTypeRegistrar ...)` — added in core v4.4
- Uses `GeneratedTypeRegistrar` interface from core `PhpOpcua\Client\Repository\`
- Every generated DTO implements `WireSerializable` so it round-trips through `opcua-session-manager`'s `ManagedClient` and through the PSR-16 cache safely (JSON-gated by `WireTypeRegistry`)
- Composer constraint: `"php-opcua/opcua-client": "^4.4.0"`

## Idiomatic patterns AI agents should follow

1. **Pass registrars to `loadGeneratedTypes()`, never call `registerCodecs()` yourself.** The builder handles topo-sort and dedup.

2. **Use the spec's `NodeIds` constants instead of magic strings.** `RoboticsNodeIds::MOTION_DEVICE_SYSTEM_TYPE` is more durable than `'ns=20;i=1004'` (the constant survives namespace remapping, the literal doesn't).

3. **Don't construct registrars with `only: true`** unless explicitly debugging. The dependency graph exists for a reason.

4. **Don't try to import every spec at once.** Each `<Spec>Registrar` you add costs codec registration time at boot. Add only the specs you actually use.

5. **DTOs are `readonly class`** — public readonly properties, not getters. `$axis->minValue`, never `$axis->getMinValue()`.

6. **Enum case names are TitleCase + underscore-separated for multi-word names.** Generator output is deterministic — `MANUAL_HIGH_SPEED`, not `MANUALHIGHSPEED` or `manualHighSpeed`.

7. **Never hand-edit files under `src/`** — they're regenerated. Edit `generate.php` if generation logic needs to change.

8. **Use `RoboticsRegistrar` etc. by class-string, not instance, in `composer.json` autoload hints** if you're listing them somewhere for IDE help (they're already PSR-4 autoloaded).

## Common pitfalls (read before generating code)

Don't write code that:

- Hard-codes `ns=N` numeric prefixes from the constants — the runtime namespace index may differ. Always pass the constant (or use `resolveNodeId('/Objects/MyMachine/…')`).
- Tries to `unserialize()` a cached DTO — they're WireSerializable, decoded via the wire registry (gadget-chain-safe).
- Calls `$client->getRepository()->register(new <Spec>Codec())` manually when a `<Spec>Registrar` exists — duplicate registration is rejected with a clear exception, but you're doing extra work for nothing.
- Imports a DTO class directly when the value comes from a `read()` — the auto-decode gives you the right type automatically.
- Edits files under `src/` — they get overwritten on next `composer generate`.
- Mixes registrars from different `opcua-client-nodeset` versions — pin the same minor version as your `opcua-client`.
- Assumes a TYPE exists if the spec's `<UADataType>` only has a `Definition` (some specs define abstract Structure types — there's no codec for those, only their concrete subtypes get codecs).

Full catalog in [`references/PITFALLS.md`](references/PITFALLS.md).

## Related packages in the php-opcua ecosystem

- **`opcua-client`** — the OPC UA client. v4.4+ required. `loadGeneratedTypes()` and `GeneratedTypeRegistrar` live there.
- **`opcua-session-manager`** — generated codecs work transparently through `ManagedClient` (the daemon registers them server-side; results decode through the wire registry).
- **`opcua-cli`** — `opcua-cli generate:nodeset <file.NodeSet2.xml>` runs the same generator against an arbitrary NodeSet2 (e.g. a vendor-specific XML you got from the manufacturer).
- **`laravel-opcua`** / **`symfony-opcua`** — framework integrations. `Opcua::connect()->loadGeneratedTypes(...)` works identically to direct `ClientBuilder` usage.
