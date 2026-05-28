# Code generation reference

Two commands handle the NodeSet2.xml ↔ PHP-code boundary:

- **`generate:nodeset <file.NodeSet2.xml>`** — XML in, PHP out. No server connection.
- **`dump:nodeset <endpoint>`** — server in, XML out. Requires a session.

Both share infrastructure: `src/CodeGenerator.php`, `src/NodeSetParser.php`, `src/NodeSetXmlBuilder.php`.

## `generate:nodeset` — turn XML into typed PHP

The same generator that produces `php-opcua/opcua-client-nodeset`, exposed as a CLI command for one-off use against vendor or custom NodeSet2.xml files.

### Quick usage

```bash
opcua-cli generate:nodeset path/to/Vendor.NodeSet2.xml \
    --output=src/Generated/OpcUa \
    --namespace='App\OpcUa\Vendor'
```

### What gets generated

Per NodeSet2.xml input, you get a per-spec directory layout under `--output`:

```
src/Generated/OpcUa/
├── VendorNodeIds.php                    ← public const NodeId strings for well-known nodes
├── VendorRegistrar.php                  ← implements PhpOpcua\Client\Repository\GeneratedTypeRegistrar
├── Enums/
│   ├── MotionStateEnum.php              ← enum: int with TitleCase cases
│   └── …
├── DataTypes/
│   ├── AxisInfo.php                     ← readonly class DTO with WireSerializable
│   └── …
└── Codecs/
    ├── AxisInfoCodec.php                ← implements PhpOpcua\Client\Encoding\ExtensionObjectCodec
    └── …
```

Identical shape to what `php-opcua/opcua-client-nodeset` ships for OPC Foundation specs.

### Options

| Option | Default | Purpose |
| --- | --- | --- |
| `--output=PATH` | `./generated/` | Output directory (created if missing) |
| `--namespace=NS` | `Generated\OpcUa` | PHP namespace for generated classes |

### Naming derivation

The spec name (used as the class prefix `<SpecName>Registrar`, `<SpecName>NodeIds`) is derived from the NodeSet2.xml's `<Model ModelUri="...">` value, after a series of well-known mappings (e.g. `http://opcfoundation.org/UA/Robotics/` → `Robotics`).

For vendor URIs the CLI falls back to the last path segment of the URI (e.g. `http://my-company.com/UA/Plc1/` → `Plc1`).

### Wiring the generated code into an application

After `composer dump-autoload` picks up the new namespace, plug the registrar in:

```php
use PhpOpcua\Client\ClientBuilder;
use App\OpcUa\Vendor\VendorRegistrar;

$client = ClientBuilder::create()
    ->loadGeneratedTypes(new VendorRegistrar())
    ->connect('opc.tcp://...');

$value = $client->read('ns=2;s=AxisInfo')->getValue();
// → App\OpcUa\Vendor\DataTypes\AxisInfo (typed DTO)
```

### Dependencies on other NodeSets

If the input NodeSet2.xml has `<RequiredModel>` references to other specs (e.g. DI, Machinery), the generator emits `dependencyRegistrars()` returns referring to **other registrars at the same namespace** (e.g. `App\OpcUa\Vendor\DIRegistrar`).

If those dependent NodeSets aren't ALSO generated, runtime will fail with a "missing dependency: DIRegistrar" error from `ClientBuilder::loadGeneratedTypes()`.

Two strategies:

1. **Generate every dependent NodeSet** the same way, in the same `--namespace` tree.
2. **Or use the shipped `php-opcua/opcua-client-nodeset`** for OPC Foundation upstream specs (DI, Machinery, etc.) — the generator detects this and emits `\PhpOpcua\Nodeset\DI\DIRegistrar` in the dependency list instead of regenerating it. Verify by inspecting the generated `<SpecName>Registrar::dependencyRegistrars()` body.

### Idempotence

Same XML → same PHP output, byte-for-byte (modulo `php-cs-fixer` formatting). Safe to re-run as part of a build pipeline.

### What it does NOT generate

- **Service / runtime code.** No connection logic, no read/write wrappers. Pure types.
- **Tests.** No PHPUnit / Pest scaffolds.
- **Composer config.** You handle `composer.json` autoload entries yourself (or rely on PSR-4 picking up the new namespace).

## `dump:nodeset` — turn server into XML

The reverse: connect to a server, walk its address space, write out a NodeSet2.xml.

### Quick usage

```bash
# Dump everything (every non-zero namespace)
opcua-cli dump:nodeset opc.tcp://192.168.1.100:4840 --output=MyPLC.NodeSet2.xml

# Dump only namespace 2 (your application's namespace)
opcua-cli dump:nodeset opc.tcp://192.168.1.100:4840 --output=MyPLC.NodeSet2.xml --namespace=2
```

### Options

| Option | Default | Purpose |
| --- | --- | --- |
| `--output=FILE` | (required) | Path to the XML file to write |
| `--namespace=N` | (all non-zero) | Restrict export to namespace index N |

### What gets exported

Per the OPC UA NodeSet2 schema:

- `<Models>` — the server's NamespaceArray, mapped to ModelUris
- `<UAObject>` / `<UAVariable>` / `<UAObjectType>` / `<UAVariableType>` / `<UADataType>` / `<UAReferenceType>` / `<UAMethod>` / `<UAView>` — every node in scope
- `<UADataType>/<Definition>` — for Structure DataTypes, the field list (so `generate:nodeset` against the output produces matching codecs)
- `<Aliases>` — the standard NodeId aliases
- `<References>` — cross-node references

The output is a valid NodeSet2.xml that should round-trip through `generate:nodeset` to produce typed PHP.

### Performance

Large servers (10k+ nodes) take minutes. The CLI walks the address space using `browse` calls, paged via continuation points. **Always set `--namespace=N`** when you know your target — it skips the OPC UA standard namespace (ns=0) which has thousands of well-known nodes already covered by `php-opcua/opcua-client-nodeset`.

### Idempotence

Two consecutive dumps of the same server produce nearly identical XML. Timestamps in node attributes (`CurrentTime` values etc.) will differ; everything else should match.

## Combined workflow — vendor server → typed PHP

```bash
# 1. Discover what the server has
opcua-cli endpoints opc.tcp://vendor.example:4840 --json | jq -r '.endpoints[].securityPolicy' | sort -u

# 2. Dump the server's vendor-specific namespace to XML
opcua-cli dump:nodeset opc.tcp://vendor.example:4840 \
    --output=Vendor.NodeSet2.xml \
    --namespace=2 \
    -u operator -p "$OPC_PASSWORD" \
    -s Basic256Sha256 -m SignAndEncrypt

# 3. Inspect the XML to verify spec coverage
head -50 Vendor.NodeSet2.xml

# 4. Generate typed PHP from it
opcua-cli generate:nodeset Vendor.NodeSet2.xml \
    --output=src/Generated/Vendor \
    --namespace='App\OpcUa\Vendor'

# 5. composer dump-autoload (or composer install if you edited composer.json)
composer dump-autoload

# 6. Use it
php -r '
require "vendor/autoload.php";
$c = (new \PhpOpcua\Client\ClientBuilder())
    ->loadGeneratedTypes(new \App\OpcUa\Vendor\VendorRegistrar())
    ->connect("opc.tcp://vendor.example:4840");
var_dump($c->read("ns=2;s=AxisInfo")->getValue());
'
```

The pattern lets you integrate against a server whose NodeSet you don't have a priori, with full typed read/write support, in under 10 minutes.

## Where this lives in `php-opcua/opcua-client-nodeset`

The shipped 51 OPC Foundation companion specs in `php-opcua/opcua-client-nodeset` are produced by the same generator (the per-package `generate.php` calls into `php-opcua/opcua-cli`'s `CodeGenerator` internally — they share code). When you `composer require php-opcua/opcua-client-nodeset`, you get the upstream specs pre-generated.

When you have a vendor NodeSet2 NOT in upstream, that's when `opcua-cli generate:nodeset` shines.
