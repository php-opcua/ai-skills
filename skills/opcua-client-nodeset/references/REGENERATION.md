# Regeneration reference

How `composer generate` works, when to re-run it, and how to add a new spec.

## TL;DR

```bash
composer generate
```

Walks `UA-Nodeset/Schema/*.NodeSet2.xml`, emits PHP into `src/<Spec>/`. Idempotent: same input → same output, byte-for-byte. Safe to run anytime.

## What `generate.php` does

The root-level script is the canonical generator. Per NodeSet2.xml file:

1. **Parse XML metadata** — namespace URI, model version, dependencies on other NodeSets
2. **Resolve spec name** — from the XML's `<Model ModelUri="...">` plus a name-mapping table
3. **Pass 1 — Enumerations** — every `<UADataType IsAbstract="false">` with `<Definition IsOptionSet="false">` and integer `<Field Value="N">` children → emits `src/<Spec>/Enums/<Name>.php`
4. **Pass 2 — Structure DataTypes** — every `<UADataType IsAbstract="false">` with `<Definition>` whose fields reference types (not enum cases) → emits `src/<Spec>/DataTypes/<Name>.php` + `src/<Spec>/Codecs/<Name>Codec.php`
5. **Pass 3 — Well-known NodeIds** — every `<UAObjectType>`, `<UADataType>`, `<UAVariableType>` at the spec's root → emits a `public const` entry in `src/<Spec>/<Spec>NodeIds.php`
6. **Pass 4 — Dependency graph** — reads `<RequiredModel>` entries from each XML, emits `dependencyRegistrars()` return values in `src/<Spec>/<Spec>Registrar.php`
7. **Pass 5 — Enum mappings** — for every `<UAVariable>` typed against an Enumeration DataType, records `[NodeId => enum-class]` in the registrar's `getEnumMappings()`
8. **Format** — runs `vendor/bin/php-cs-fixer` over `src/` (PSR-12) so the output passes the project's lint check

Output is deterministic. CI compares `composer generate && git diff --exit-code` to ensure committed PHP matches generator output.

## When to re-generate

| Trigger | Action |
| --- | --- |
| Upstream `UA-Nodeset` released a new version | `cd UA-Nodeset && git pull && cd ..` then `composer generate` |
| You added a new spec's XML to `UA-Nodeset/Schema/` | `composer generate` |
| You modified `generate.php` itself | `composer generate` (twice — first run produces output, second confirms idempotence) |
| CI fails with "drift detected" | Pull the change, `composer generate`, commit. Don't hand-edit the diff. |

## Adding a new companion spec

1. Drop the NodeSet2 XML into `UA-Nodeset/Schema/`:
   ```bash
   cd UA-Nodeset/Schema
   curl -O https://raw.githubusercontent.com/OPCFoundation/UA-Nodeset/latest/<Spec>/Opc.Ua.<Spec>.NodeSet2.xml
   ```
2. Run the generator:
   ```bash
   cd ..  # back to repo root
   composer generate
   ```
3. Inspect the diff:
   ```bash
   git status                                       # should show a new src/<Spec>/ directory
   git diff src/<Spec>/                             # spot-check the generated files
   ```
4. Test (if you have a server that speaks the spec):
   ```php
   $client = ClientBuilder::create()
       ->loadGeneratedTypes(new \PhpOpcua\Nodeset\<Spec>\<Spec>Registrar())
       ->connect('opc.tcp://…');
   ```
5. Commit:
   ```bash
   git add UA-Nodeset/Schema/Opc.Ua.<Spec>.NodeSet2.xml src/<Spec>/
   git commit -m "[NEW] <Spec> companion spec"
   ```

If the new spec depends on other specs not yet in this repo, the generator will emit a clear "missing dependency: `<DepSpec>`" error. Add the dependency XML first.

## CI verification

`.github/workflows/tests.yml` (project-level) runs:

```bash
composer install
composer generate
git diff --exit-code
```

If `composer generate` produces a diff, CI fails. The fix is always to commit the regenerated files — never disable the check.

## Vendor / proprietary NodeSets

The bundled `UA-Nodeset/` contains only the OPC Foundation upstream. For vendor-specific NodeSets (Siemens, Beckhoff, Kuka, Rockwell, etc.) **don't add them to `UA-Nodeset/`**. Instead, use the per-application generator from `php-opcua/opcua-cli`:

```bash
opcua-cli generate:nodeset /path/to/Vendor.NodeSet2.xml \
    --output=src/Generated \
    --namespace='App\OpcUa\Vendor'
```

This produces the same DTO/Codec/Registrar shape but under your application's namespace. The generated registrar implements the same `GeneratedTypeRegistrar` interface and plugs into `loadGeneratedTypes()` alongside the shipped ones.

Pros:
- Vendor NodeSets stay in your repo, versioned with your app
- This package's `src/` stays canonical OPC Foundation-only
- Upgrading this package doesn't risk overwriting vendor code

## How the XML is vendored

`UA-Nodeset/` is either a git submodule pointing at `OPCFoundation/UA-Nodeset` or a vendored copy (depending on how this repo was set up). To check:

```bash
git -C UA-Nodeset rev-parse HEAD                   # current upstream commit
cat .gitmodules                                    # submodule config (if submodule)
```

To update:

```bash
# Submodule
git submodule update --remote UA-Nodeset
# Or vendored
cd UA-Nodeset && git pull && cd ..
```

Then `composer generate` and commit.

## Idempotence guarantees

The generator promises:

- **Stable case ordering** — enum cases appear in the order their `<Field Value="N">` entries do in the XML
- **Stable property ordering** — DTO fields match the order of `<Field>` children in the XML
- **Stable file content** — re-running with the same XML produces byte-identical output (modulo `php-cs-fixer` formatting, which is itself deterministic)
- **Stable NodeId values** — the `ns=N` in NodeIds constants matches the XML's `NamespaceTable`

The only non-determinism is the upstream XML itself — an OPC Foundation revision that reorders fields will produce a diff. That's intended.

## Troubleshooting

| Error | Cause | Fix |
| --- | --- | --- |
| `Missing dependency: <Spec>` | The new spec's `<RequiredModel>` references a spec not in `UA-Nodeset/` | Add the dependency XML first |
| `Conflicting enum case: <NAME>` | Two XML fields produce the same PHP case name after TitleCase conversion | Rare — usually a typo in the XML; report upstream |
| `Cannot generate codec for abstract type: <Name>` | Spec has `<UADataType IsAbstract="true">` with no concrete subtypes | Expected behavior — abstract types don't get codecs |
| `Drift detected in CI` | Output PHP doesn't match what the current `generate.php` produces | Run `composer generate` locally, commit the diff |
| PHP fatal during generation | `php-cs-fixer` config drift or missing PHP extension | Run `composer install`, ensure PHP >= 8.2 with ext-dom |

## Don't hand-edit

Files under `src/` carry `@generated` annotations in the PHPDoc. Next `composer generate` overwrites them silently. If a behavior needs to change:

- **For all specs**: edit `generate.php` (the emitter logic)
- **For one spec only**: edit the upstream XML in `UA-Nodeset/Schema/<file>.NodeSet2.xml` (will diverge from upstream — track the diff carefully) or use a custom-namespace overlay via `opcua-cli generate:nodeset`

Pull requests that edit `src/` directly will fail CI's `git diff --exit-code` check.
