# Custom modules through the daemon

`opcua-client` lets you register custom `ServiceModule` instances on the `ClientBuilder`. The session-manager makes those custom methods reachable from `ManagedClient` too, so the framework / facade / autowired client never needs to know about the bridge.

## The mechanic

1. **Daemon side** — when the daemon's internal `Client` is built, custom modules are passed via the daemon's constructor config. Every session in the daemon shares the same set of registered modules.
2. **Client side** — `ManagedClient::__call($method, $args)` falls through to the daemon's generic `invoke` IPC command if `$method` isn't on the built-in 51-method whitelist BUT IS a registered custom method.
3. **Param transport** — args travel through the JSON wire codec. The daemon decodes them using the `ParamDeserializerInterface` chain (`BuiltInParamDeserializer` first, then any custom deserializer registered via `CommandHandler::registerParamDeserializer()`).
4. **Result transport** — return values are serialized by `TypeSerializer` (every WireSerializable DTO is registered in the daemon's `WireTypeRegistry`).

## Worked example — adding a `ping` module

### 1. Define the module

```php
namespace App\OpcUa;

use PhpOpcua\Client\Module\ServiceModule;
use PhpOpcua\Client\Kernel\ClientKernelInterface;
use PhpOpcua\Client\Protocol\SessionService;
use PhpOpcua\Client\Wire\WireTypeRegistry;

final class PingModule extends ServiceModule
{
    public function name(): string { return 'ping'; }
    public function requires(): array { return []; }

    public function register(ClientKernelInterface $kernel, SessionService $session): array
    {
        return [
            'ping' => fn (): PingResult => new PingResult(
                alive: $kernel->getConnectionState()->name === 'Connected',
                rttMs: 0,                                    // measure your way
            ),
        ];
    }

    public function registerWireTypes(WireTypeRegistry $registry): void
    {
        $registry->register(PingResult::class);
    }
}

final readonly class PingResult implements \PhpOpcua\Client\Wire\WireSerializable {
    public function __construct(public bool $alive, public int $rttMs) {}
    public static function wireTypeId(): string { return 'PingResult'; }
    public function jsonSerialize(): array { return ['alive' => $this->alive, 'rttMs' => $this->rttMs]; }
    public static function fromWireArray(array $d): static { return new self((bool) ($d['alive'] ?? false), (int) ($d['rttMs'] ?? 0)); }
}
```

### 2. Register the module on the daemon

```php
use PhpOpcua\SessionManager\Daemon\SessionManagerDaemon;

$daemon = new SessionManagerDaemon(
    socketPath: '/var/run/opcua/sm.sock',
    clientModules: [new PingModule()],                       // applied to every session
);
$daemon->run();
```

### 3. Call from ManagedClient

```php
use PhpOpcua\SessionManager\Client\ManagedClient;
use App\OpcUa\PingResult;

$client = new ManagedClient();
$client->connect('opc.tcp://localhost:4840');

/** @var PingResult $result */
$result = $client->ping();
var_dump($result->alive, $result->rttMs);
```

The IDE may not autocomplete `ping()` because it's not on the `OpcUaClientInterface` — that's expected. Either:

- Use `$client->invokeRemote('ping', [])` for the explicit form
- Or define an interface that extends `OpcUaClientInterface` and adds your custom methods, then `@var` the variable to it

## Custom parameter types

If your module's `register()` callable accepts a non-built-in param type (a custom DTO), tell the daemon how to decode it from JSON:

```php
namespace App\OpcUa;

use PhpOpcua\SessionManager\Serialization\ParamDeserializerInterface;

final class JobDeserializer implements ParamDeserializerInterface
{
    public function supports(string $methodName, int $paramIndex, mixed $rawValue): bool
    {
        return $methodName === 'submitJob' && $paramIndex === 0 && is_array($rawValue);
    }

    public function deserialize(string $methodName, int $paramIndex, mixed $rawValue): mixed
    {
        return new Job(
            id: $rawValue['id'],
            payload: $rawValue['payload'],
        );
    }
}
```

Register it on the daemon (NOT on the client — the deserializer runs daemon-side, decoding the JSON envelope):

```php
$daemon = new SessionManagerDaemon(
    /* ... */
    clientModules: [new JobModule()],
    paramDeserializers: [new JobDeserializer()],
);
```

Or programmatically against an already-built daemon:

```php
$daemon->getCommandHandler()->registerParamDeserializer(new JobDeserializer());
```

The registry consults deserializers **first-match-wins** in registration order. `BuiltInParamDeserializer` is always last (handles NodeId, Variant, DataValue, BuiltinType, etc.).

## Custom result types

If your module returns a custom DTO (like `PingResult` above), register it on the daemon's wire registry inside the module's `registerWireTypes()` method (called by `ServiceModule::register()` flow). No client-side registration needed — the IPC envelope carries the `__t` discriminator and `TypeSerializer` looks it up on both sides.

## Limits

- A custom method **cannot** override a built-in (`read`, `browse`, etc.) — `ModuleConflictException` raised at daemon boot.
- Custom methods that mutate global daemon state (e.g. `setSomething`) are still blocked by the whitelist — setters aren't reachable via IPC. If you need configuration, do it at daemon construction time.
- Parameters and results MUST be wire-serializable — anything that can't be encoded as JSON-allowlist payload raises `SerializationException` at the IPC boundary.

## When to use custom modules vs application code

| Use custom module | Use application code |
| --- | --- |
| Per-session OPC UA state (cursor, cache, monitored-item list, retry budget) | Stateless data transformation |
| Logic that needs `ClientKernelInterface` (raw `send` / `receive`, low-level retry) | Anything reachable via standard `OpcUaClientInterface` methods |
| Cross-cutting concerns reused by every session (audit logging, custom subscription policy) | Single-call ad-hoc scripts |

In short: if the logic needs to live next to the OPC UA session, make it a module. If it's a transformation over results, keep it in your application.
