# Session Manager

The `opcua-session-manager` daemon runs alongside your Symfony app. Without it, every client call opens a fresh TCP connection (handshake, secure channel, session, activation — 200-500ms). With it, all calls share a daemon-side session pool over IPC.

## Architecture

```
php-fpm-1 ─┐
php-fpm-2 ─┼─► IPC ─► opcua-session-manager daemon ─► TCP ─► OPC UA server
messenger ─┘
            ↑
            "shared session pool, one TCP connection per server"
```

PHP-FPM workers, Messenger handlers, and console commands all share the same daemon-side sessions.

## Starting the daemon

```bash
php bin/console opcua:session
```

Options:

| Option | Default | Effect |
|---|---|---|
| `--timeout=600` | `session_manager.timeout` | Session inactivity timeout (s) |
| `--cleanup-interval=30` | `session_manager.cleanup_interval` | Sweep cadence (s) |
| `--max-sessions=100` | `session_manager.max_sessions` | Concurrent session cap |
| `--socket-mode=0600` | `session_manager.socket_mode` | Unix socket perms (octal) |

Note: unlike the Laravel command, `opcua:session` here does **not** expose `--log-channel` / `--cache-store` CLI options. The logger and cache pool are wired at container compile time from the YAML config.

## systemd setup (Linux production)

```ini
# /etc/systemd/system/opcua-session.service
[Unit]
Description=OPC UA Session Manager (Symfony)
After=network-online.target redis.service
Wants=network-online.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/app
EnvironmentFile=/var/www/app/.env.local
ExecStart=/usr/bin/php /var/www/app/bin/console opcua:session
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
KillSignal=SIGTERM
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opcua-session
journalctl -u opcua-session -f
```

## Supervisor setup

```ini
; /etc/supervisor/conf.d/opcua-session.conf
[program:opcua-session]
process_name=%(program_name)s
command=/usr/bin/php /var/www/app/bin/console opcua:session
autostart=true
autorestart=true
startsecs=5
startretries=3
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/supervisor/opcua-session.log
stopwaitsecs=30
stopsignal=TERM
```

## Docker / Compose

Run the daemon as a sidecar container that shares a volume with the application:

```yaml
# compose.prod.yml
services:
  app:
    build: ./
    volumes:
      - opcua-sock:/var/run/opcua
    environment:
      OPCUA_SOCKET_PATH: unix:///var/run/opcua/session-manager.sock

  opcua-session:
    build: ./
    command: php bin/console opcua:session
    volumes:
      - opcua-sock:/var/run/opcua
    environment:
      OPCUA_SOCKET_PATH: unix:///var/run/opcua/session-manager.sock
    restart: unless-stopped

volumes:
  opcua-sock:
```

In YAML, point `socket_path` to the shared volume:

```yaml
php_opcua_symfony_opcua:
    session_manager:
        socket_path: '%env(OPCUA_SOCKET_PATH)%'
```

## How `shouldUseSessionManager()` decides

```php
// OpcuaManager::shouldUseSessionManager() pseudo-code
if (! $config['enabled']) return false;
$unixPath = TransportFactory::toUnixPath($config['socket_path']);
return $unixPath !== null
    ? file_exists($unixPath)
    : true;  // TCP endpoint: assume present, fail on first IPC call if not
```

Implication: under Unix, if the daemon dies, the app transparently falls back to direct `Client` mode after the socket file disappears. Under TCP loopback, the app keeps trying IPC until `TcpLoopbackTransport` raises `DaemonException`. Catch that to fail over gracefully.

## Auth token

Set `OPCUA_AUTH_TOKEN` to a long random string (≥ 32 bytes). Both daemon and clients read it from the same env via YAML:

```yaml
php_opcua_symfony_opcua:
    session_manager:
        auth_token: '%env(OPCUA_AUTH_TOKEN)%'
```

```env
OPCUA_AUTH_TOKEN=...
```

Without it, anyone with filesystem/TCP access to the IPC endpoint can issue commands.

## Health check

```bash
# Unix socket — daemon should be holding it
test -S /var/run/opcua-session-manager.sock && echo "Socket present"

# Programmatic
php bin/console debug:container --tag=php-opcua  # services should show
```

Via the manager:
```php
$opcuaManager->isSessionManagerRunning();  // bool
```

For richer health info (session count, memory), use `opcua-cli session-manager:stats`:
```bash
opcua-cli session-manager:stats --socket=/var/run/opcua-session-manager.sock
```

## Auto-publish lifecycle

When `auto_publish: true`:

1. Daemon `boot()` loads the bundle config and for each connection with `auto_connect: true`:
   - Opens a session
   - Creates each entry in `subscriptions[]`
   - Registers each entry's `monitored_items[]` and `event_monitored_items[]`
2. Daemon spins a `PublishService` that calls `publish()` periodically per session
3. Notifications are converted to PSR-14 events and dispatched through the `EventDispatcherInterface` (Symfony's event dispatcher, injected via bundle)
4. `#[AsEventListener]`-tagged listeners receive the events — sync or via Messenger

Manual `publish()` is blocked while auto-publish is active.

## Symfony Messenger for heavy listener work

Listeners that do DB I/O / HTTP / mailing should hand off to Messenger. Tag a Message:

```php
namespace App\Message;

final class SensorReadingReceived
{
    public function __construct(
        public readonly int $clientHandle,
        public readonly float $value,
        public readonly \DateTimeImmutable $sampledAt,
    ) {}
}
```

Hand off from the listener (which stays cheap and synchronous):

```php
namespace App\EventListener;

use App\Message\SensorReadingReceived;
use PhpOpcua\Client\Event\DataChangeReceived;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Messenger\MessageBusInterface;

#[AsEventListener]
final class DispatchSensorReading
{
    public function __construct(private MessageBusInterface $bus) {}

    public function __invoke(DataChangeReceived $event): void
    {
        $this->bus->dispatch(new SensorReadingReceived(
            clientHandle: $event->clientHandle,
            value: (float) $event->dataValue->getValue(),
            sampledAt: $event->dataValue->sourceTimestamp ?? new \DateTimeImmutable(),
        ));
    }
}
```

Now the actual DB work lives in a Messenger handler — back-pressure isolated from the daemon's publish loop.

## Graceful daemon restart

`SIGTERM` triggers:

1. Stop accepting new IPC connections
2. Wait for in-flight requests to complete (`stopwaitsecs=30` is enough)
3. Close every daemon-side session via `CloseSessionRequest` so the server doesn't accumulate ghosts
4. Unlink the Unix socket file
5. Exit 0

After a restart, calls hit a momentary `DaemonException` until the daemon comes back up. Configure `auto_retry` on connections that need to survive a restart.

## When NOT to run the daemon

- **CLI-only console commands** that connect once, do work, exit. The TCP overhead is dominated by your work, the daemon adds setup cost.
- **Single-worker Messenger consumer** with low OPC UA throughput. One persistent connection inside the long-lived worker process is enough.
- **CI/test environments.** Use `MockClient` or direct `Client` with a Docker test server.

## Mixed-version deployments

The daemon and the Symfony bundle versions **must** match (within a minor). The daemon's command handlers know a specific set of methods — a v4.4 application calling `historyInsertData` against a v4.3 daemon gets `BadMethodCallException`.

Upgrade order:
1. `composer require php-opcua/opcua-session-manager:^4.4` on the daemon host
2. Restart the daemon (`sudo systemctl restart opcua-session`)
3. `composer require php-opcua/symfony-opcua:^4.4` on the application
4. Redeploy
