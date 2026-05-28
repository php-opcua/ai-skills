# Serialization reference

Everything that crosses the IPC boundary travels as JSON. Two layers handle it:

- **`WireMessageCodec`** (`src/Ipc/WireMessageCodec.php`) — envelope framing (NDJSON), size/depth limits, type-discriminator (`__t`) allowlist
- **`TypeSerializer`** (`src/Serialization/TypeSerializer.php`) — OPC UA-aware encoder/decoder for `NodeId`, `Variant`, `DataValue`, `ExtensionObject`, and every WireSerializable DTO

## Envelope format

```jsonc
// Command (client → daemon)
{
  "v": 1,                         // protocol version
  "id": "<uuid>",                 // correlation ID
  "cmd": "invoke",                // command name (open, close, invoke, ping, etc.)
  "args": {
    "session": "<session-id>",    // when applicable
    "method": "read",
    "params": [/* JSON-encoded args */]
  }
}

// Result (daemon → client)
{
  "v": 1,
  "id": "<uuid>",                 // matches the command's id
  "result": { "__t": "DataValue", /* fields */ }
}

// Error (daemon → client)
{
  "v": 1,
  "id": "<uuid>",
  "error": {
    "type": "SessionNotFoundException",
    "message": "Session abc... not found",
    "code": 404
  }
}
```

One JSON object per line (`\n`-delimited NDJSON). Framing is handled by `AbstractStreamTransport` regardless of the underlying transport.

## Limits enforced by WireMessageCodec

| Limit | Value | Behaviour |
| --- | --- | --- |
| Max envelope size | 16 MiB | Larger frame → `SerializationException` |
| Max nesting depth | 32 levels | Deeper structure → `SerializationException` |
| Binary mode | enforced | No charset detection / conversion |
| Allowed `__t` types | registered set only | Unknown discriminator → `SerializationException` |

The size cap matters in practice for large `historyReadRaw` responses — if you fetch a million data points in one call, the envelope can exceed 16 MiB. Solution: page via `numValuesPerNode` or chunk via `startTime` / `endTime` windows.

## Type discriminator (`__t`)

Every non-primitive value is wrapped:

```jsonc
{ "__t": "NodeId",       "v": "ns=2;s=Temp" }
{ "__t": "Variant",      "type": 6, "value": 42 }
{ "__t": "DataValue",    "variant": {...}, "status": 0, "sourceTs": "...", "serverTs": "..." }
{ "__t": "BrowseNode",   "reference": {...}, "children": [...] }
```

The discriminator is the `wireTypeId()` of the corresponding PHP class. `TypeSerializer` looks it up in `WireTypeRegistry` (a static allowlist) and reconstructs the typed instance via `fromWireArray()`. **Unknown discriminators are rejected** — no `unserialize()`, no gadget chain, no class instantiation by attacker-controlled name.

## What's serializable out of the box

Registered by `CoreWireTypes::register()` (called from `WireTypeRegistry` constructor):

- `NodeId`, `QualifiedName`, `LocalizedText`
- `DataValue`, `Variant`, `ExtensionObject`
- `BrowseNode`, `ReferenceDescription`
- `EndpointDescription`, `UserTokenPolicy`
- Enums: `BuiltinType`, `NodeClass`, `BrowseDirection`, `ConnectionState`

Each `ServiceModule` adds its own result DTOs via `registerWireTypes()`:

- ReadWrite: `CallResult`
- Browse: `BrowseResultSet`
- Subscription: `SubscriptionResult`, `MonitoredItemResult`, `PublishResult`, `TransferResult`
- TranslateBrowsePath: `BrowsePathResult`, `BrowsePathTarget`
- NodeManagement: `AddNodesResult`
- ServerInfo: `BuildInfo`
- History: `HistoryUpdateResult` (v4.4.0)
- Aggregate: `Interval`, `AggregateFunction` (enum), `AggregateOptions` (v4.4.0)

## What's NOT serializable

- Anything not on the `__t` allowlist
- PHP closures
- Resource handles (open sockets, file descriptors)
- Non-`WireSerializable` objects

If you try to send one of these, `SerializationException` raises at the encode boundary with a clear "no codec for type X" message.

## Sensitive fields

`SessionConfig::SENSITIVE_FIELDS` (v4.2.0) lists fields the daemon strips from in-memory state after the OPC UA session is established:

- `password`
- `clientKeyPath`
- `caCertPath`
- `userKeyPath`

Method `SessionConfig::sanitized()` returns a clone with these fields blanked — useful for logging the config without leaking secrets. The CommandHandler always uses `sanitized()` before passing config to PSR-3 log calls.

The fields are still transmitted on the wire during the `open` command (that's how the daemon receives them). To protect that wire:

- Local IPC (Unix socket with 0600 mode, or TCP loopback) keeps the wire off the network.
- Auth token gate prevents unauthorized connections to the IPC.
- Application-level: pass `password` as `null` and use a credential lookup module on the daemon side to fetch from a vault.

## Custom param deserialization

The IPC `invoke` command passes `params` as an array of JSON values. The daemon needs to know how to turn each raw value into a typed PHP object before calling the actual method.

`ParamDeserializerInterface` is the contract:

```php
interface ParamDeserializerInterface
{
    public function supports(string $methodName, int $paramIndex, mixed $rawValue): bool;
    public function deserialize(string $methodName, int $paramIndex, mixed $rawValue): mixed;
}
```

`ParamDeserializerRegistry` consults registered deserializers **first-match-wins** in registration order. `BuiltInParamDeserializer` (registered last by default) handles the common cases:

- `NodeId|string` parameters
- `BuiltinType` enum values
- `DataValue` parameters
- `Variant` parameters
- `DateTimeImmutable` (ISO 8601 strings)
- Arrays of any of the above

Custom modules register their own deserializers via `CommandHandler::registerParamDeserializer()` — see [`CUSTOM-MODULES.md`](CUSTOM-MODULES.md) for an example.

## Bypass — sending raw JSON

If you're debugging or building a test harness, you can talk to the daemon's IPC directly with `curl` / `nc`:

```bash
# Unix socket
echo '{"v":1,"id":"test","cmd":"ping"}' | nc -U /var/run/opcua/sm.sock
# {"v":1,"id":"test","result":true}
```

Always single-line JSON, always `\n` terminator, always answer in <16 MiB.

## What the wire is NOT

- **It is not RPC** — there's no remote method invocation in the classic sense. The daemon dispatches via a static table.
- **It is not a serialization format library** — `WireMessageCodec` is intentionally minimal. No type coercion, no schema migration, no versioning negotiation beyond the `v` integer.
- **It is not stable across major versions** — v4.x → v5.x will change `v` and break wire compatibility. Plan coordinated upgrades.
