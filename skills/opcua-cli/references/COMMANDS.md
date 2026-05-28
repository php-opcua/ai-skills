# Commands reference

The 11 commands with exact flags and JSON output shape. Every command shares the [global options](../SKILL.md#global-options-every-command) and (when connecting) the [security options](../SKILL.md#security-options-every-connection-aware-command).

## `browse <endpoint> [<nodeId>]`

List references from a node. Default starting node is `i=85` (Objects folder).

```bash
opcua-cli browse opc.tcp://localhost:4840
opcua-cli browse opc.tcp://localhost:4840 'i=85'
opcua-cli browse opc.tcp://localhost:4840 /Objects/MyPLC
opcua-cli browse opc.tcp://localhost:4840 'ns=2;i=1000'
opcua-cli browse opc.tcp://localhost:4840 /Objects --recursive --depth=3
opcua-cli browse opc.tcp://localhost:4840 /Objects --json
```

**Specific options**: `--recursive`, `--depth=N` (default 3 when recursive)

**Output (human)**: tree with `▸` markers for foldable subtrees, `─` for leaves, columns aligned for displayName / nodeClass / nodeId.

**Output (`--json`)**:
```json
{
  "node": "ns=0;i=85",
  "references": [
    { "nodeId": "ns=0;i=2253", "browseName": "Server", "displayName": "Server", "nodeClass": "Object", "isForward": true }
  ]
}
```

When `--recursive`, each reference may carry a nested `children` array.

## `read <endpoint> <nodeId>`

Read an attribute of a node. Default is `Value`.

```bash
opcua-cli read opc.tcp://localhost:4840 'i=2259'
opcua-cli read opc.tcp://localhost:4840 'ns=2;i=1001' --attribute=DisplayName
opcua-cli read opc.tcp://localhost:4840 'ns=2;i=1001' --json
```

**Specific options**: `--attribute=NAME` — one of `Value`, `DisplayName`, `BrowseName`, `DataType`, `NodeClass`, `Description`, `AccessLevel`, `NodeId`

**Output (human)**: single-line `value (statusCode) @ sourceTimestamp` or, for non-Value attributes, the typed display.

**Output (`--json`)**:
```json
{
  "node": "i=2259",
  "attribute": "Value",
  "value": 0,
  "statusCode": 0,
  "type": "Int32",
  "sourceTimestamp": "2026-05-28T10:30:45.123456Z",
  "serverTimestamp": "2026-05-28T10:30:45.124000Z"
}
```

## `write <endpoint> <nodeId> <value>`

Write a value to a node.

```bash
opcua-cli write opc.tcp://localhost:4840 'ns=2;i=1001' 42 --type=Int32
opcua-cli write opc.tcp://localhost:4840 'ns=2;i=2000' true --type=Boolean
opcua-cli write opc.tcp://localhost:4840 'ns=2;i=2000' '"hello world"' --type=String
opcua-cli write opc.tcp://localhost:4840 'ns=2;i=1001' 42                   # auto-detect type via read-before-write
```

**Specific options**: `--type=TYPE` — one of `Boolean`, `SByte`, `Byte`, `Int16`, `UInt16`, `Int32`, `UInt32`, `Int64`, `UInt64`, `Float`, `Double`, `String`. Without `--type`, the CLI reads the node's DataType attribute first (one extra round trip).

**Value parsing**: integer types parse via PHP's `(int)`; `Double` / `Float` via `(float)`; `Boolean` accepts `true`/`false`/`1`/`0`; `String` is the literal arg (quote in shell if it contains spaces).

**Output (human)**: `OK` or `Failed: <StatusCode>`. Exit code 0 on `Good`, 4 on any `Bad*` status.

**Output (`--json`)**:
```json
{ "node": "ns=2;i=1001", "value": 42, "type": "Int32", "statusCode": 0, "result": "Good" }
```

## `watch <endpoint> <nodeId>`

Stream value changes. Subscription mode by default (server pushes via OPC UA Subscription). `--interval=N` switches to polling (calls `read` every N ms).

```bash
opcua-cli watch opc.tcp://localhost:4840 'ns=2;i=1001'                       # subscription mode
opcua-cli watch opc.tcp://localhost:4840 'ns=2;i=1001' --interval=250        # polling every 250 ms
opcua-cli watch opc.tcp://localhost:4840 'ns=2;i=1001' --json | jq -c '.value'
```

**Specific options**: `--interval=N` (milliseconds — switches to polling)

**Termination**: Ctrl-C. The command catches SIGINT and closes the subscription / disconnects cleanly.

**Output (human)**: one line per change: `[2026-05-28T10:30:45.123456Z] value=42.5 status=Good`.

**Output (`--json`)**: one JSON object per line (NDJSON, suitable for streaming jq):
```json
{"timestamp":"2026-05-28T10:30:45.123Z","value":42.5,"type":"Double","statusCode":0,"sourceTimestamp":"2026-05-28T10:30:45.123Z"}
{"timestamp":"2026-05-28T10:30:45.623Z","value":42.7,"type":"Double","statusCode":0,"sourceTimestamp":"2026-05-28T10:30:45.623Z"}
```

## `explore <endpoint>`

Full-screen TUI (Linux/macOS only — Windows not yet supported by upstream `php-tui/php-tui`).

```bash
opcua-cli explore opc.tcp://localhost:4840
opcua-cli explore opc.tcp://localhost:4840 -u operator -p operator123
```

See [`EXPLORE.md`](EXPLORE.md) for the full key bindings and layout.

**Rejects `--json` and `--debug`** (would corrupt the TUI). Use `--debug-stderr` or `--debug-file=PATH` to capture debug output without disturbing the display.

## `endpoints <endpoint>`

Discover the server's endpoints — list every (URL, securityPolicy, securityMode, userTokenPolicy) tuple. No session opened — pure GetEndpoints discovery.

```bash
opcua-cli endpoints opc.tcp://localhost:4840
opcua-cli endpoints opc.tcp://localhost:4840 --json | jq '.endpoints[].securityPolicy'
```

**Output (human)**: table of endpoints with security policies and user token policies.

**Output (`--json`)**:
```json
{
  "endpoints": [
    {
      "endpointUrl": "opc.tcp://localhost:4840",
      "securityPolicy": "http://opcfoundation.org/UA/SecurityPolicy#None",
      "securityMode": "None",
      "userIdentityTokens": [
        { "tokenType": "Anonymous", "policyId": "open62541-anonymous-policy" }
      ],
      "transportProfileUri": "http://opcfoundation.org/UA-Profile/Transport/uatcp-uasc-uabinary",
      "securityLevel": 0
    }
  ]
}
```

## `generate:nodeset <file.NodeSet2.xml>`

Generate PHP classes from a NodeSet2.xml file. No server connection.

```bash
opcua-cli generate:nodeset path/to/Opc.Ua.Di.NodeSet2.xml \
    --output=src/Generated/Di \
    --namespace='App\OpcUa\Di'
```

**Specific options**:
- `--output=PATH` — output directory (default `./generated/`)
- `--namespace=NS` — PHP namespace for generated classes (default `Generated\OpcUa`)

**Generates** (same shape as `php-opcua/opcua-client-nodeset`):
- `<SpecName>NodeIds.php` — public const NodeId strings
- `<SpecName>Registrar.php` — implements `GeneratedTypeRegistrar`
- `Enums/<Name>.php` — PHP `enum: int`
- `DataTypes/<Name>.php` — `readonly class` DTOs
- `Codecs/<Name>Codec.php` — `ExtensionObjectCodec` implementations

See [`CODEGEN.md`](CODEGEN.md) for the full workflow + how the generated code integrates with `opcua-client`'s `loadGeneratedTypes()`.

## `dump:nodeset <endpoint>`

Read a server's address space and export as NodeSet2.xml.

```bash
opcua-cli dump:nodeset opc.tcp://192.168.1.100:4840 --output=MyPLC.NodeSet2.xml
opcua-cli dump:nodeset opc.tcp://192.168.1.100:4840 --output=MyPLC.NodeSet2.xml --namespace=2
```

**Specific options**:
- `--output=FILE` — required, the XML file to write
- `--namespace=N` — restrict export to namespace index N (default: all non-zero namespaces)

**Performance**: a large server (10k+ nodes) can take minutes. Always filter with `--namespace` to scope the dump.

## `trust <endpoint>`

Connect and capture the server's certificate into the user trust store (TOFU — Trust On First Use).

```bash
opcua-cli trust opc.tcp://server:4840
opcua-cli trust opc.tcp://server:4840 --trust-store=/etc/opcua/trust --trust-policy=fingerprint+expiry
```

**Specific options**:
- `--trust-store=PATH` (default: `$HOME/.opcua`)
- `--trust-policy=POLICY` — `fingerprint`, `fingerprint+expiry`, `full` (CA chain). Default: `fingerprint`.
- `--no-trust-policy` — disable policy enforcement (accept any cert). Dev only.

**Output (human)**: confirmation that the cert was added, with its thumbprint.

**Output (`--json`)**:
```json
{ "endpoint": "opc.tcp://server:4840", "thumbprint": "ab:cd:12:34:...", "addedAt": "2026-05-28T10:30:45Z" }
```

## `trust:list`

List trusted certificates from the trust store. No server connection.

```bash
opcua-cli trust:list
opcua-cli trust:list --trust-store=/etc/opcua/trust --json
```

**Output (human)**: table of (thumbprint, subject, expiresAt, endpointUrls).

**Output (`--json`)**:
```json
{
  "trustStore": "/home/user/.opcua",
  "certificates": [
    { "thumbprint": "ab:cd:12:34:...", "subject": "CN=…", "issuer": "…", "notAfter": "2027-01-01T00:00:00Z", "endpointUrls": ["opc.tcp://server:4840"] }
  ]
}
```

## `trust:remove <thumbprint>`

Remove a trusted certificate by thumbprint. No server connection.

```bash
opcua-cli trust:remove ab:cd:12:34:5678:90...
opcua-cli trust:remove ab:cd:12:34:5678:90 --trust-store=/etc/opcua/trust
```

**Exit codes**: 0 if removed, 1 if no matching thumbprint, 2 if trust store unreadable.

## Help

Every command supports `--help` / `-h`:

```bash
opcua-cli --help                                                  # overall help
opcua-cli read --help                                             # per-command help
```

`--version` / `-v` prints the CLI version (matches `php-opcua/opcua-cli` package version, lock-step with `opcua-client`).
