# Pitfalls

Common mistakes when working with `symfony-opcua`. Each entry has the **smell**, **why it's wrong**, and the **fix**.

## P1 — Forgetting the bundle registration

If Flex didn't run (or you cloned an older app), the bundle is not in `config/bundles.php` and autowiring fails:

```
Cannot autowire service "App\...": argument "$opcua" of method "__construct()"
references interface "PhpOpcua\Client\OpcUaClientInterface" but no such service exists.
```

Fix — add to `config/bundles.php`:

```php
return [
    // ...
    PhpOpcua\SymfonyOpcua\PhpOpcuaSymfonyOpcuaBundle::class => ['all' => true],
];
```

## P2 — Calling `disconnect()` in HTTP requests with the daemon enabled

```php
// ❌ controller
public function show()
{
    $value = $this->opcua->read('i=2259')->getValue();
    $this->opcua->disconnect();  // tears down daemon session
    return $this->json($value);
}
```

Under `ManagedClient`, `disconnect()` is a daemon-side close — next request pays full connect overhead again. Let the daemon manage lifecycle.

```php
// ✓
return $this->json($this->opcua->read('i=2259')->getValue());
```

## P3 — Manual `publish()` under `auto_publish`

```php
// ❌
$this->opcua->publish();  // returns auto_publish_active error, no notifications
```

The daemon's auto-publish loop owns the publish cycle. Subscribe to events instead:

```php
// ✓
#[AsEventListener]
final class OnChange { public function __invoke(DataChangeReceived $e): void { /* ... */ } }
```

## P4 — Long-running worker without the daemon

`OpcuaManager` is a public singleton. In Messenger workers or FrankenPHP worker mode without the daemon, the singleton holds a direct `Client` whose connection state lives between request invocations. Subscriptions, cached metadata, etc. leak.

Fix: enable the daemon, or reset on `kernel.terminate` (Messenger doesn't fire this — use `WorkerStoppedEvent` or per-message logic instead).

## P5 — `auto_accept: true` in production

```yaml
# ❌
php_opcua_symfony_opcua:
    connections:
        default:
            auto_accept: true
```

TOFU bypass. MITM on first contact gets permanently trusted. Use `auto_accept: true` only during bootstrap, then flip to `false` after `opcua-cli trust` seeds the store.

## P6 — Plaintext password with `security_mode: None`

```yaml
# ❌
connections:
    default:
        security_mode: None
        username: op
        password: '%env(OPCUA_PASSWORD)%'
```

Username token uses the server's cert to encrypt the password. With `None`, server behavior is implementation-specific — may send plaintext. Use `Sign` or `SignAndEncrypt` whenever credentials are supplied.

## P7 — Mixed daemon + app versions

App on v4.4, daemon on v4.3 → `BadMethodCallException` the first time the app calls `historyInsertData` / `openFile` / `aggregate`. Upgrade daemon first, then app.

## P8 — Hot-loop reads inside a Messenger handler

```php
// ❌
public function __invoke(StartPollingMessage $msg): void
{
    while (true) {
        $v = $this->opcua->read('ns=2;s=Temp')->getValue();
        sleep(1);
    }
}
```

The Messenger handler blocks the worker forever. Use subscriptions + auto-publish, or schedule a tick via `Symfony\Scheduler` that triggers a short-lived handler:

```php
// ✓ — see INTEGRATIONS.md → Symfony Scheduler section
```

## P9 — Synchronous heavy work in event listener

```php
// ❌
#[AsEventListener]
final class StoreOnChange
{
    public function __invoke(DataChangeReceived $e): void
    {
        $this->em->persist(/* ... */);
        $this->em->flush();           // hits Postgres
        $this->mailer->send(/* ... */); // hits SMTP
    }
}
```

The daemon's publish loop blocks while the listener runs. At >50 events/sec, you back-pressure the loop and drop notifications. Hand off to Messenger:

```php
// ✓ — listener dispatches a Message; Messenger handler does the I/O
$this->bus->dispatch(new SensorReadingReceived(/* ... */));
```

## P10 — Cron probe without timeout/overlap protection

```php
// ❌ scheduler
RecurringMessage::every('60 seconds', new OpcuaProbe());
```

If the PLC hangs for 90s, two probes stack; for 5min, five. Configure Messenger handler with explicit timeout (and `--time-limit` on the consumer to recycle workers):

```php
public function __invoke(OpcuaProbe $msg): void
{
    // Use the per-call timeout in the connection config; default 5s
}
```

## P11 — Trust store in an ephemeral path

```yaml
# ❌ Docker without persistent volume
trust_store_path: /tmp/opcua-trust
```

Container restart → trust store wiped → all reconnects fail. Use `%kernel.project_dir%/var/opcua-trust-store` with a Docker volume mount.

## P12 — Catching `\Exception` and swallowing

```php
// ❌
try { $this->opcua->write('ns=2;s=Setpoint', 42.5); }
catch (\Exception) { /* shrug */ }
```

Writes can return Good with side-effects (e.g., clamping) or Bad without an exception (status code on the return value). Always check return value or specific exceptions:

```php
// ✓
$status = $this->opcua->write('ns=2;s=Setpoint', 42.5);
if (! \PhpOpcua\Client\Types\StatusCode::isGood($status)) {
    throw new \RuntimeException("Write failed: " . \PhpOpcua\Client\Types\StatusCode::getName($status));
}
```

## P13 — Caching `$opcuaManager->connection('x')` in a property forever

```php
// ❌ singleton service
public function __construct(OpcuaManager $mgr) {
    $this->client = $mgr->connection('plc-1');  // held forever
}
```

If the daemon dies / socket changes, your service holds a dead handle. Resolve lazily:

```php
// ✓
public function __construct(private OpcuaManager $mgr) {}
private function client(): OpcUaClientInterface { return $this->mgr->connection('plc-1'); }
```

The manager caches internally and reconnects on demand.

## P14 — `connectTo` without `as:` in a request loop

```php
// ❌
foreach ($endpoints as $url) {
    $this->mgr->connectTo($url)->read('i=2259');  // new client each iteration
}
```

Supply `as: $url` to dedupe within the request:

```php
// ✓
foreach ($endpoints as $url) {
    $this->mgr->connectTo($url, as: $url)->read('i=2259');
}
```

## P15 — Daemon owned by root + web user can't read socket

```bash
# ❌
sudo php bin/console opcua:session  # daemon owns socket as root
```

`socket_mode: 0600` + root-owned = web user can't read. Run the daemon as the web user:

```ini
# ✓ systemd
User=www-data
Group=www-data
```

## P16 — Resolving `OpcUaClientInterface` in a `kernel.request` listener that runs before container compile

Rare, but: listeners with priority > 250 may run before subscribed services are ready. Inject `ContainerInterface` and resolve lazily, or use service decoration.

## P17 — `OPCUA_AUTH_TOKEN` hardcoded in YAML

```yaml
# ❌
session_manager:
    auth_token: hardcoded-secret-in-git
```

Committed to repo. Always:

```yaml
# ✓
auth_token: '%env(OPCUA_AUTH_TOKEN)%'
```

## P18 — Treating `subscriptionId` / `monitoredItemId` as global identifiers

Both are server-scoped. After a daemon restart with `transferSubscriptions` recovery, ids may change. Don't store them long-term; use `client_handle` for application-level correlation.

## P19 — Using `MockClient` in feature tests without container override

```php
// ❌
$mock = MockClient::create()->onRead(...);
$client = static::createClient();
$client->request('GET', '/');  // hits real config, ignores mock
```

You created the mock but didn't tell the container to use it:

```php
// ✓
self::getContainer()->set(OpcUaClientInterface::class, $mock);
$client = static::createClient();
```

## P20 — `read_metadata_cache: true` with frequently-changing address space

The cache assumes node types are stable. If you have NodeSets that change at runtime, enabling the cache returns stale `BuiltinType` and `auto_detect_write_type` writes the wrong format. Either disable, or invalidate explicitly: `$opcua->invalidateCache('ns=2;s=DynamicTag')`.

## P21 — Confusing `OpcUaClientInterface` autowire with `connection($name)`

```php
// ❌
public function __construct(
    private OpcUaClientInterface $client,  // resolves to DEFAULT connection only
) {}
```

If you need named connections, inject `OpcuaManager`:

```php
// ✓
public function __construct(private OpcuaManager $mgr) {}
public function handle(): void { $this->mgr->connection('plc-2')->read(...); }
```

## P22 — `cache_pool: cache.app` in serverless / Vercel-style deployment

`cache.app` defaults to filesystem adapter. Serverless environments lose the cache between cold starts. Configure a Redis pool explicitly when caching is performance-critical:

```yaml
framework:
    cache:
        pools:
            opcua.cache:
                adapter: cache.adapter.redis
                provider: '%env(REDIS_URL)%'

php_opcua_symfony_opcua:
    session_manager:
        cache_pool: opcua.cache
```
