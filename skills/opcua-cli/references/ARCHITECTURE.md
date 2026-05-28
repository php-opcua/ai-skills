# Architecture reference

Internal wiring of `opcua-cli`. Read this when contributing a new command, replacing the output adapter, or debugging unexpected argv parsing.

## Top-level layout

```
opcua-cli/
├── bin/opcua-cli                   ← PHP entry script; loads autoload + Application
├── src/
│   ├── Application.php             ← registers commands, parses argv, runs the right one
│   ├── ArgvParser.php              ← custom argv parser (no Symfony Console)
│   ├── CommandRunner.php           ← builds Client (security/auth/trust/timeout), dispatches
│   ├── CodeGenerator.php           ← NodeSet2.xml → PHP code emitter
│   ├── NodeSetParser.php           ← NodeSet2.xml parser (DOM-based)
│   ├── NodeSetXmlBuilder.php       ← Address space → NodeSet2.xml builder
│   ├── StreamLogger.php            ← minimal PSR-3 logger writing to a stream
│   ├── Commands/
│   │   ├── CommandInterface.php    ← contract every command implements
│   │   ├── BrowseCommand.php
│   │   ├── ReadCommand.php
│   │   ├── WriteCommand.php
│   │   ├── WatchCommand.php
│   │   ├── ExploreCommand.php
│   │   ├── EndpointsCommand.php
│   │   ├── GenerateNodesetCommand.php
│   │   ├── DumpNodesetCommand.php
│   │   ├── TrustCommand.php
│   │   ├── TrustListCommand.php
│   │   └── TrustRemoveCommand.php
│   ├── Output/
│   │   ├── OutputInterface.php     ← write/error/table/tree/value
│   │   ├── ConsoleOutput.php        ← coloured, padded, tree-drawing
│   │   └── JsonOutput.php           ← machine-readable
│   └── Tui/
│       ├── ExploreApp.php          ← the explore command's TUI app (php-tui)
│       └── TreeNode.php            ← tree-node model for the TUI
├── box.json.dist                   ← Box (PHAR) builder config for release-binaries.yml
└── composer.json                   ← php-opcua/opcua-client ^4.4 + php-tui/php-tui ^0.2.1
```

## Why no Symfony Console

Original constraint: keep runtime deps minimal. `php-opcua/opcua-client` requires only `ext-openssl` + PSR interfaces. Symfony Console would pull in `symfony/string`, `symfony/process`, etc. — a 15+ MiB dependency for an argv parser.

`ArgvParser` is ~250 lines of focused argv handling:

- Positional args (in order, named by the command)
- Long options (`--foo=bar`, `--foo bar`, `--foo`)
- Short options (`-f bar`, `-f`, clustered `-abc`)
- Long-only `--help`, `--version`, `--json`
- Global vs command-specific options
- Error messages with the unknown / malformed token quoted

It's not a generic argv library — it specifically serves `opcua-cli`'s needs. Bug fixes here affect every command's CLI parsing.

## How a command runs (end-to-end)

Take `opcua-cli read opc.tcp://server:4840 'i=2259' --json`:

1. **`bin/opcua-cli`** — minimal PHP wrapper: `require autoload`, instantiate `Application`, call `run($argv)`, exit with the returned code.

2. **`Application::run($argv)`**:
   - Registers all 11 commands into a class-keyed map
   - First positional arg (`read`) → selects `ReadCommand`
   - Selects output adapter: `--json` → `JsonOutput`, else `ConsoleOutput`
   - Parses argv via `ArgvParser` against the command's option schema
   - Delegates to `CommandRunner::run($command, $parsedArgs, $output)`

3. **`CommandRunner`**:
   - If the command needs a connection (`$command->requiresConnection()`):
     - Builds a `Client` via `ClientBuilder::create()`
     - Applies `-s` / `-m` / `--cert` / `--key` / `--ca` / `-u` / `-p` / `-t`
     - Applies `--trust-store` / `--trust-policy`
     - Calls `$client->connect($endpoint)`
   - Calls `$command->execute($parsedArgs, $client, $output)`
   - On exception: maps to exit code (1/2/3/4 depending on type), logs to `--debug-*`, returns
   - Disconnects the client

4. **`ReadCommand::execute()`**:
   - Reads the value via `$client->read($nodeId, $attribute)`
   - Calls `$output->value($dataValue)` to emit
   - Returns `0` on `Good`, `4` on Bad status

5. **`JsonOutput::value()`** serializes the `DataValue` to the documented JSON shape and writes to stdout.

## `CommandInterface`

```php
namespace PhpOpcua\Cli\Commands;

interface CommandInterface
{
    public function getName(): string;                 // 'read'
    public function getDescription(): string;          // one-liner shown in --help
    public function getUsage(): string;                // 'read <endpoint> <nodeId>' shown in errors
    public function requiresConnection(): bool;        // false for generate:nodeset, trust:list, trust:remove
    public function execute(
        array $args,
        ?OpcUaClientInterface $client,
        OutputInterface $output,
    ): int;
}
```

`$client` is `null` when `requiresConnection() === false`.

## `OutputInterface`

```php
namespace PhpOpcua\Cli\Output;

interface OutputInterface
{
    public function write(string $message): void;       // generic text
    public function error(string $message): void;       // stderr or error styling
    public function value(DataValue $dv): void;         // a single Variant value with metadata
    public function tree(array $references): void;      // browse results
    public function table(array $rows): void;           // tabular data
    // … and a few more
}
```

`ConsoleOutput` emits human-readable text with ANSI escape codes (colours, padding, tree-drawing).
`JsonOutput` accumulates into a structured array and writes one final JSON document on flush (or one object per call for NDJSON streaming).

Both share the same interface so command code is output-format-agnostic.

## TUI

`Tui/ExploreApp.php` builds on `php-tui/php-tui` (^0.2.1). The TUI is a separate concern from the rest of the CLI:

- Loaded only when `explore` is invoked (not on `require autoload` — `php-tui` classes are autoload-resolved lazily)
- Manages its own render loop (`while (!$shouldQuit) { tick(); render(); read_input(); }`)
- Coordinates with the rest of `opcua-cli` via `OpcUaClientInterface` (same client surface as scripted commands)

`TreeNode.php` is a small data structure: each node holds the OPC UA `ReferenceDescription`, the loaded child list (lazy), and the expanded/collapsed state.

## `CodeGenerator` & `NodeSetParser` / `NodeSetXmlBuilder`

The three classes that power `generate:nodeset` and `dump:nodeset`:

- **`NodeSetParser`** — DOM-based XML parser, walks `<UADataType>` / `<UAEnumeration>` / `<UAObject>` / `<UAVariable>` etc. Returns a typed in-memory representation.
- **`CodeGenerator`** — takes the parsed representation, emits PHP source code (Enums, DataTypes, Codecs, Registrar, NodeIds). Deterministic ordering, runs `php-cs-fixer` if available, writes files atomically.
- **`NodeSetXmlBuilder`** — reverse direction: given a connected `Client`, browses the address space, builds an in-memory representation, writes it as NodeSet2.xml. Used by `dump:nodeset`.

The same code is reused by `php-opcua/opcua-client-nodeset`'s `generate.php` — that package's `composer generate` script delegates to `opcua-cli`'s `CodeGenerator` rather than re-implementing it.

## Distribution

Three ways the CLI reaches users:

1. **Composer dep** (`composer require php-opcua/opcua-cli`) → `vendor/bin/opcua-cli` symlinks to `bin/opcua-cli`.
2. **Global composer** (`composer global require …`) → `~/.composer/vendor/bin/opcua-cli`.
3. **PHAR / standalone binary** — built by `release-binaries.yml` workflow using `humbug/box`. `box.json.dist` defines the PHAR layout (bin entry, included files, compression). The PHAR includes a bundled PHP runtime via `appimage` / similar.

The workflow runs on tag pushes and uploads PHARs as GitHub Release assets. The same CLI source produces all three forms — no per-form code.

## Testing

- `tests/Unit/` — argv parsing, output formatting, code generation against fixture XML
- `tests/Integration/` — end-to-end against `uanetstandard-test-suite`'s servers (Linux only)

The `composer test` script runs Pest. CI runs both groups against PHP 8.2/8.3/8.4/8.5 × Linux/macOS/Windows for unit, Linux only for integration.

## Logging

`StreamLogger` is a minimal PSR-3 impl writing log entries to a stream (`STDOUT` for `--debug`, `STDERR` for `--debug-stderr`, a file handle for `--debug-file=PATH`).

Log lines look like:
```
[2026-05-28T10:30:45Z] DEBUG client.read nodeId=ns=2;s=Temp duration=12ms
```

When no debug flag is set, the logger receives `NullLogger` (zero overhead).

## Why this architecture

- **Single-binary distribution** is possible because there's no plugin loader, no DI container, no event bus. Application directly instantiates everything.
- **Trivial extensibility** for commands: `class FooCommand implements CommandInterface` + register in `Application::registerCommands()`. ~20 lines for a new command.
- **Output strategy pattern** keeps every command output-format-agnostic. `JsonOutput` shape is stable per command.
- **No global state** — `Application` is the only stateful object, instantiated fresh on each invocation. Multiple CLI invocations in the same shell are entirely isolated.
