# Registrars reference

How `GeneratedTypeRegistrar` works, what each registrar does, and how the dependency graph resolves at boot.

## Contract (from `php-opcua/opcua-client`)

```php
namespace PhpOpcua\Client\Repository;

interface GeneratedTypeRegistrar
{
    public function registerCodecs(ExtensionObjectRepository $repository): void;

    /** @return self[] */
    public function dependencyRegistrars(): array;

    /** @return array<string, class-string<\BackedEnum>> */
    public function getEnumMappings(): array;
}
```

Every per-spec registrar in this package implements that interface. Their structure is identical apart from the codec list, dependency list, and enum map — pure data.

## How `loadGeneratedTypes()` resolves them

`ClientBuilder::loadGeneratedTypes(GeneratedTypeRegistrar ...$rs)` does this:

1. Collects each `$r` and recursively expands `$r->dependencyRegistrars()` into a flat list
2. Deduplicates by class name (first one wins — by-design, registrars are pure data)
3. Topologically sorts the list so dependencies come before dependents
4. For each registrar in topo order:
   - Calls `registerCodecs($this->extensionObjectRepository)` — every codec is registered into the per-client repository
   - Calls `getEnumMappings()` — entries are merged into the client's enum-mapping table; later mappings for the same NodeId override earlier ones (but in practice mappings never collide; the generator guarantees uniqueness)
5. The fully-populated client is returned

## The dependency graph (excerpt)

A simplified view of the most common transitive dependencies:

```
DI ──► (foundation — most specs depend on it)
 ├── Machinery ──► (process / asset / state machines)
 │    ├── MachineTool
 │    │    ├── CuttingTool
 │    │    └── …
 │    ├── Robotics
 │    ├── Tightening
 │    ├── LADS
 │    ├── CNC
 │    └── …
 ├── BACnet
 ├── FDI / FDT
 ├── AutoID
 ├── PROFINET
 ├── PROFIBUS
 ├── IO-Link (IA)
 └── …
```

In practice, ~80 % of registrars transitively depend on `DI` (Device Integration). The Machinery family adds the asset / state-machine layer most modern industrial specs build on.

## Common usage patterns

### Single domain — let dependencies auto-resolve

```php
$builder->loadGeneratedTypes(new RoboticsRegistrar());
// → DI + Machinery + Robotics codecs registered.
```

This is the canonical pattern. `dependencyRegistrars()` does the work; you just name the leaf spec.

### Multiple independent branches

```php
$builder->loadGeneratedTypes(
    new RoboticsRegistrar(),                                 // → DI + Machinery
    new BACnetRegistrar(),                                   // → DI (shared, deduped)
);
// Shared base specs registered once. DI codecs not duplicated.
```

Useful when one application integrates with multiple vendor stacks (a robot via Robotics, a building HVAC controller via BACnet).

### Full kitchen sink (rare)

```php
$builder->loadGeneratedTypes(
    new RoboticsRegistrar(),
    new MachineToolRegistrar(),
    new MachineVisionRegistrar(),
    new BACnetRegistrar(),
    new PackMLRegistrar(),
    new MTConnectRegistrar(),
    // …
);
```

Every additional registrar costs boot-time codec registration (microseconds per codec — but they add up across the 193 total codecs). Add only what you use.

### `only: true` — debug knob

```php
$builder->loadGeneratedTypes(new RoboticsRegistrar(only: true));
// → ONLY Robotics codecs. DI / Machinery NOT registered.
```

Reading a Robotics Structure DataType that references `DI/EngineeringUnit` will then decode the EngineeringUnit field as a raw `ExtensionObject` (no codec for it). Useful when isolating a bug ("does Robotics decode correctly without Machinery polluting the picture?"). **Don't use in production.**

## How to know which registrar to use

The OPC Foundation publishes a NodeSet2.xml per spec. Each spec has a known stable name. The mapping between spec name and PHP namespace is one-to-one and matches the upstream:

| OPC Foundation NodeSet | PHP namespace | Registrar class |
|---|---|---|
| Opc.Ua.Robotics.NodeSet2.xml | `PhpOpcua\Nodeset\Robotics` | `RoboticsRegistrar` |
| Opc.Ua.MachineTool.NodeSet2.xml | `PhpOpcua\Nodeset\MachineTool` | `MachineToolRegistrar` |
| Opc.Ua.Machinery.NodeSet2.xml | `PhpOpcua\Nodeset\Machinery` | `MachineryRegistrar` |
| Opc.Ua.DI.NodeSet2.xml | `PhpOpcua\Nodeset\DI` | `DIRegistrar` |
| Opc.Ua.BACnet.NodeSet2.xml | `PhpOpcua\Nodeset\BACnet` | `BACnetRegistrar` |
| Opc.Ua.AutoID.NodeSet2.xml | `PhpOpcua\Nodeset\AutoID` | `AutoIDRegistrar` |
| Opc.Ua.MachineVision.NodeSet2.xml | `PhpOpcua\Nodeset\MachineVision` | `MachineVisionRegistrar` |
| Opc.Ua.PackML.NodeSet2.xml | `PhpOpcua\Nodeset\PackML` | `PackMLRegistrar` |
| Opc.Ua.PROFINET.NodeSet2.xml | `PhpOpcua\Nodeset\PROFINET` | `PROFINETRegistrar` |
| Opc.Ua.MTConnect.NodeSet2.xml | `PhpOpcua\Nodeset\MTConnect` | `MTConnectRegistrar` |
| … | … | … |

See [`CATALOGUE.md`](CATALOGUE.md) for the full list with one-line domain descriptions.

## Detecting which spec a server uses

```php
// Read the server's NamespaceArray (i=2255) — list of namespace URIs in order
$namespaces = $client->read('i=2255')->getValue();
// → e.g. ['http://opcfoundation.org/UA/', 'http://opcfoundation.org/UA/Robotics/', 'http://my-vendor.com/UA/']

// Each URI matches a known nodeset namespace (or a vendor extension)
```

The companion-spec URIs follow the pattern `http://opcfoundation.org/UA/<SpecName>/`. Match against the catalogue to pick the right registrar(s).

## When to write your own registrar

Almost never. The exceptions:

- A vendor ships a NodeSet2.xml that's not in `UA-Nodeset/` upstream → use `opcua-cli generate:nodeset <vendor.NodeSet2.xml>` to produce a one-off registrar under your application's namespace (not `PhpOpcua\Nodeset\`). The generated class also implements `GeneratedTypeRegistrar`.
- You want to override a single codec for testing (`replaceModule()` pattern on the core).

In both cases, instantiate your custom registrar and pass it to `loadGeneratedTypes()` alongside (or instead of) the shipped ones.

## Inspecting a registrar at runtime

```php
$registrar = new RoboticsRegistrar();

$dependencies = $registrar->dependencyRegistrars();
foreach ($dependencies as $dep) {
    echo $dep::class . "\n";
}

$enumMap = $registrar->getEnumMappings();
foreach ($enumMap as $nodeIdStr => $enumClass) {
    echo "$nodeIdStr → $enumClass\n";
}
```

Useful for debugging "why isn't this enum auto-cast?" or "which codecs got registered?". Each list is plain data — no side effects.
