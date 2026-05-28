# Configuration

Single YAML file: `config/packages/php_opcua_symfony_opcua.yaml`. Three top-level keys: `default`, `session_manager`, `connections`.

The bundle reads it through `DefinitionConfigurator` (modern `AbstractBundle::configure()`), so `php bin/console config:dump-reference php_opcua_symfony_opcua` shows the full tree.

## `default`

Name of the connection used when `$opcuaManager->connect()` is called without an argument, and the one returned when `OpcUaClientInterface` is autowired.

```yaml
php_opcua_symfony_opcua:
    default: default
```

## `session_manager`

Daemon-related settings — IPC endpoint, session lifecycle, daemon-process logging/cache.

| Key | Default | Notes |
|---|---|---|
| `enabled` | `true` | If false, `shouldUseSessionManager()` returns false unconditionally → always direct `Client`. |
| `socket_path` | `%kernel.project_dir%/var/opcua-session-manager.sock` | IPC URI: `unix://<path>`, `tcp://127.0.0.1:<port>` (loopback only), or scheme-less path (= `unix://`). On Windows, override to `tcp://127.0.0.1:9990`. |
| `timeout` | `600` | Session inactivity timeout (s). |
| `cleanup_interval` | `30` | Sweep idle sessions every N seconds. |
| `auth_token` | `null` | Shared secret for IPC. Recommended in production. |
| `max_sessions` | `100` | Hard cap on concurrent daemon-side sessions. |
| `socket_mode` | `0600` | Unix socket permission bits. |
| `allowed_cert_dirs` | `[]` | Whitelist of directories the daemon may load `client_certificate` / `client_key` from. `[]` = no restriction. |
| `log_channel` | `null` | Monolog channel name. `null` → default `logger`. `opcua` → `monolog.logger.opcua`. |
| `cache_pool` | `cache.app` | Symfony PSR-6 cache pool service ID. Bundle wraps it in `Psr16Cache`. |
| `auto_publish` | `false` | Daemon auto-publishes for sessions with subscriptions, dispatches PSR-14 events through Symfony. See `EVENTS.md`. |

### `socket_path` form selection

| Form | Example | Used by |
|---|---|---|
| `unix://<path>` | `unix:///var/run/opcua.sock` | Linux, macOS |
| `tcp://127.0.0.1:<port>` | `tcp://127.0.0.1:9990` | Windows |
| scheme-less path | `%kernel.project_dir%/var/opcua-session-manager.sock` | Backwards compat → same as `unix://` |

`TcpLoopbackTransport` enforces loopback-only on both daemon and client. Anything non-127.0.0.1/::1 is refused.

### Why these defaults make sense

- `timeout=600s` — survives an idle PHP-FPM/Messenger worker between requests but releases server resources within 10 minutes of inactivity.
- `cleanup_interval=30s` — small enough that ghost sessions don't accumulate, large enough not to thrash on busy daemons.
- `socket_mode=0600` — file perm is the only access control on local socket; lock it down by default.
- `cache_pool=cache.app` — uses the framework's filesystem-default cache; safe even before you customize cache config.

## `connections`

Map of `name => connection-config`. The default connection key is `default`.

### Endpoint and security

| Key | Default | Notes |
|---|---|---|
| `endpoint` | (required) | `opc.tcp://host:port`. Use `%env(OPCUA_ENDPOINT)%`. |
| `security_policy` | `None` | `None`, `Basic128Rsa15`, `Basic256`, `Basic256Sha256`, `Aes128Sha256RsaOaep`, `Aes256Sha256RsaPss`, `ECC_nistP256`, `ECC_nistP384`, `ECC_brainpoolP256r1`, `ECC_brainpoolP384r1`. |
| `security_mode` | `None` | `None`, `Sign`, `SignAndEncrypt`. |
| `username` | `null` | Username for `UserName` token. |
| `password` | `null` | Use `%env(OPCUA_PASSWORD)%`. |
| `client_certificate` | `null` | Path to PEM or DER. Auto-generated in-memory if omitted and a policy is set. |
| `client_key` | `null` | Path to private key. |
| `ca_certificate` | `null` | CA cert for server cert validation. |
| `user_certificate` | `null` | X.509 user token cert. |
| `user_key` | `null` | X.509 user token key. |

### Client behaviour

| Key | Default | Notes |
|---|---|---|
| `timeout` | `5.0` | Network timeout (s) per request. |
| `auto_retry` | `null` | Max reconnection retries on transient failure. `null` = don't retry. |
| `batch_size` | `null` | Max items per service request. `null` = use server's `MaxNodesPerRead/Write`. |
| `browse_max_depth` | `10` | Default `browseRecursive()` depth. |

### Trust store

| Key | Default | Notes |
|---|---|---|
| `trust_store_path` | `null` | Directory for trusted/rejected server certs. Recommended: `%kernel.project_dir%/var/opcua-trust-store`. |
| `trust_policy` | `null` | `fingerprint`, `fingerprint+expiry`, `full`. |
| `auto_accept` | `false` | TOFU on first contact. Dev only. |
| `auto_accept_force` | `false` | Re-trust previously rejected certs. Almost never right. |

### Type and metadata

| Key | Default | Notes |
|---|---|---|
| `auto_detect_write_type` | `true` | Read-before-write detects the OPC UA type. One extra round-trip per first-time write per node. |
| `read_metadata_cache` | `false` | Cache node DataType metadata so auto-detect doesn't pay the round-trip every time. Set to `true` when the address space is stable. |

### Per-connection logging (v4.3+)

| Key | Notes |
|---|---|
| `log_channel` | Name of a Monolog channel. Resolved lazily via the bundle's `ServiceLocator`. Falls back to default `logger` when missing. |

Logger resolution priority:
1. Runtime override via `$opcuaManager->setLogger()` / `useConsoleLogger()`
2. Per-connection config `'logger'` (PSR-3 instance) — unused via YAML, useful when overriding at runtime
3. Per-connection config `'log_channel'` (Monolog channel name)
4. Application default `LoggerInterface`

### Auto-connect / declarative subscriptions

```yaml
php_opcua_symfony_opcua:
    session_manager:
        auto_publish: true
    connections:
        plc-1:
            endpoint: '%env(PLC1_ENDPOINT)%'
            auto_connect: true
            subscriptions:
                - publishing_interval: 500.0
                  max_keep_alive_count: 5
                  lifetime_count: 2400
                  monitored_items:
                      - { node_id: 'ns=2;s=Temperature', client_handle: 1, sampling_interval: 250.0, queue_size: 1 }
                  event_monitored_items:
                      - node_id: 'i=2253'
                        client_handle: 10
                        select_fields: ['EventId', 'EventType', 'SourceName', 'Time', 'Message', 'Severity']
```

`client_handle` is the application-side correlation key for `DataChangeReceived` / `EventNotificationReceived` events.

## Defining multiple connections

```yaml
php_opcua_symfony_opcua:
    default: plc-1
    connections:
        plc-1:
            endpoint: '%env(PLC1_ENDPOINT)%'
            security_policy: Basic256Sha256
            security_mode: SignAndEncrypt
            username: '%env(PLC1_USER)%'
            password: '%env(PLC1_PASS)%'
        plc-2:
            endpoint: '%env(PLC2_ENDPOINT)%'
        historian:
            endpoint: '%env(HISTORIAN_ENDPOINT)%'
            browse_max_depth: 3
            timeout: 30.0
```

```php
$opcuaManager->connection('plc-1')->read('ns=2;s=Temp');
$opcuaManager->connection('historian')->historyReadRaw(...);
```

`OpcUaClientInterface` autowire still resolves to `plc-1` (the default).

## Per-call config override (ad-hoc)

```php
$client = $opcuaManager->connectTo(
    'opc.tcp://discovered.example:4840',
    [
        'security_policy' => 'Basic256Sha256',
        'security_mode' => 'SignAndEncrypt',
        'username' => 'guest',
        'password' => '',
    ],
    as: 'discovered-1',
);
```

Without `as:`, the connection is throwaway. With `as:`, it's cached in the manager's connections array.

## Environment variable cheat-sheet

```env
# Connection
OPCUA_ENDPOINT=opc.tcp://server:4840
OPCUA_USERNAME=operator
OPCUA_PASSWORD=changeme
OPCUA_CLIENT_CERT=/etc/opcua/client.pem
OPCUA_CLIENT_KEY=/etc/opcua/client.key

# Session manager
OPCUA_AUTH_TOKEN=long-random-shared-secret
```

YAML referencing:
```yaml
php_opcua_symfony_opcua:
    session_manager:
        auth_token: '%env(OPCUA_AUTH_TOKEN)%'
    connections:
        default:
            endpoint: '%env(OPCUA_ENDPOINT)%'
            username: '%env(OPCUA_USERNAME)%'
            password: '%env(OPCUA_PASSWORD)%'
            client_certificate: '%env(OPCUA_CLIENT_CERT)%'
            client_key: '%env(OPCUA_CLIENT_KEY)%'
```

## Dedicated cache pool

```yaml
# config/packages/cache.yaml
framework:
    cache:
        pools:
            opcua.cache:
                adapter: cache.adapter.filesystem

# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    session_manager:
        cache_pool: opcua.cache
```

The bundle wraps `opcua.cache` in `Symfony\Component\Cache\Psr16Cache` and injects it into `OpcuaManager`.

## Dedicated Monolog channel

```yaml
# config/packages/monolog.yaml
monolog:
    channels: ['opcua']
    handlers:
        opcua:
            type: rotating_file
            path: '%kernel.logs_dir%/opcua.log'
            channels: ['opcua']

# config/packages/php_opcua_symfony_opcua.yaml
php_opcua_symfony_opcua:
    session_manager:
        log_channel: opcua
```

The bundle's `LoggerResolverFactory` maps the channel name to `monolog.logger.opcua` at container compile time.

## Common config mistakes

- **Path without `%kernel.project_dir%`** for `socket_path` or `trust_store_path` → CI / Docker breaks because the path resolves elsewhere.
- **`auto_accept: true` in production.** TOFU bypass on first contact. Fine for dev, never for prod.
- **`auto_detect_write_type: true` without `read_metadata_cache: true`** — doubles every first-time-write round-trip.
- **TCP loopback socket_path with non-loopback host** — refused by the transport layer.
- **Forgetting `OPCUA_AUTH_TOKEN`** — anyone with filesystem/TCP access to the IPC endpoint can issue commands.
