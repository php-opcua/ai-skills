# Deployment reference

Standing up the SessionManagerDaemon in production.

## Manual start

```bash
php bin/opcua-session-manager [options]
```

The process foregrounds. Logs to stderr by default. SIGTERM / SIGINT triggers graceful shutdown — all sessions close cleanly, the socket file is removed, then the process exits 0.

## CLI options (parsed by `src/Cli/ArgvParser.php`)

| Option | Default | Purpose |
| --- | --- | --- |
| `--socket=<uri>` | `/tmp/opcua-session-manager.sock` (Linux/macOS), `tcp://127.0.0.1:<auto-port>` (Windows) | IPC endpoint. Accepts `unix:///abs/path` or `tcp://127.0.0.1:<port>` |
| `--timeout=<seconds>` | `600` | Per-session inactivity before automatic cleanup |
| `--cleanup-interval=<seconds>` | `30` | How often the cleanup timer scans `SessionStore` |
| `--auth-token=<string>` | none | IPC auth token (visible to `ps` — prefer `--auth-token-file` or `OPCUA_AUTH_TOKEN` env) |
| `--auth-token-file=<path>` | none | Read auth token from file (chmod 600) |
| `--max-sessions=<int>` | `100` | Hard cap on concurrent sessions; further `open` commands raise |
| `--socket-mode=<octal>` | `0600` | Unix socket file permission (ignored on TCP) |
| `--allowed-cert-dirs=<comma-list>` | `none` | Whitelist of cert directories. Sessions trying to load a cert outside these dirs are refused. Default `none` means no cert loading allowed at all. |

Auth-token resolution priority: `OPCUA_AUTH_TOKEN` env → `--auth-token-file` → `--auth-token`. The first non-empty value wins.

## systemd unit

```ini
# /etc/systemd/system/opcua-session-manager.service
[Unit]
Description=OPC UA Session Manager (php-opcua/opcua-session-manager)
Documentation=https://www.php-opcua.com/documentation/opcua-session-manager
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=opcua
Group=opcua
WorkingDirectory=/srv/opcua-app
EnvironmentFile=/etc/opcua/sm.env

ExecStart=/usr/bin/php /srv/opcua-app/vendor/bin/opcua-session-manager \
    --socket=/var/run/opcua/sm.sock \
    --timeout=1800 \
    --cleanup-interval=60 \
    --auth-token-file=/etc/opcua/sm.token \
    --max-sessions=200 \
    --socket-mode=0660 \
    --allowed-cert-dirs=/etc/opcua/certs

# Graceful shutdown — daemon traps SIGTERM and closes every session
KillSignal=SIGTERM
TimeoutStopSec=30

Restart=on-failure
RestartSec=5

# Hardening
ProtectSystem=strict
ReadWritePaths=/var/run/opcua /var/log/opcua
PrivateTmp=true
NoNewPrivileges=true
ProtectHome=true
ProtectKernelTunables=true
ProtectControlGroups=true

[Install]
WantedBy=multi-user.target
```

`/etc/opcua/sm.env`:
```dotenv
# Read by EnvironmentFile=
OPCUA_AUTH_TOKEN_FILE=/etc/opcua/sm.token
```

Enable + start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opcua-session-manager
sudo systemctl status opcua-session-manager
journalctl -u opcua-session-manager -f
```

## Supervisor

```ini
; /etc/supervisor/conf.d/opcua-session-manager.conf
[program:opcua-session-manager]
command=/usr/bin/php /srv/opcua-app/vendor/bin/opcua-session-manager
  --socket=/var/run/opcua/sm.sock
  --auth-token-file=/etc/opcua/sm.token
directory=/srv/opcua-app
user=opcua
autostart=true
autorestart=true
stopwaitsecs=30
stopsignal=TERM
stderr_logfile=/var/log/opcua/sm.stderr.log
stdout_logfile=/var/log/opcua/sm.stdout.log
environment=OPCUA_AUTH_TOKEN_FILE="/etc/opcua/sm.token"
```

`stopsignal=TERM` is required — Supervisor's default `INT` works too because the daemon traps both, but `TERM` is the more conventional production choice.

## Docker

```dockerfile
# Dockerfile.session-manager
FROM php:8.4-cli-alpine

RUN apk add --no-cache linux-headers $PHPIZE_DEPS \
    && pecl install pcntl-stable \
    && docker-php-ext-enable pcntl \
    && apk del $PHPIZE_DEPS

# install opcua-session-manager via composer
COPY composer.json composer.lock /app/
WORKDIR /app
RUN composer install --no-dev --no-interaction --optimize-autoloader

EXPOSE 4870/tcp
# pick a fixed port for predictable container networking
CMD ["php", "vendor/bin/opcua-session-manager", \
     "--socket=tcp://0.0.0.0:4870", \
     "--auth-token-file=/run/secrets/sm_token", \
     "--max-sessions=200", \
     "--timeout=1800"]
```

> **Security caveat**: binding to `0.0.0.0` is **rejected** at construction by the loopback-only guard. The Docker pattern is to expose `tcp://0.0.0.0:4870` INSIDE the container and rely on Docker's network isolation (only the app container is on the bridge network, no port mapping to the host). If you need cross-container IPC, bind the daemon to `127.0.0.1` and use a Docker `network_mode: service:` trick to share the loopback. For most setups, a shared volume + Unix socket is cleaner:

```yaml
# docker-compose.yml
services:
  opcua-sm:
    build: { dockerfile: Dockerfile.session-manager }
    volumes:
      - opcua-sock:/var/run/opcua
      - ./certs:/etc/opcua/certs:ro
    environment:
      OPCUA_AUTH_TOKEN_FILE: /run/secrets/sm_token
    secrets: [sm_token]
    command: >
      php vendor/bin/opcua-session-manager
        --socket=unix:///var/run/opcua/sm.sock
        --socket-mode=0660
        --allowed-cert-dirs=/etc/opcua/certs

  app:
    image: my-app:latest
    volumes:
      - opcua-sock:/var/run/opcua:ro     # mount socket read-only
    environment:
      OPCUA_SOCKET: unix:///var/run/opcua/sm.sock
    depends_on: [opcua-sm]

volumes:
  opcua-sock:

secrets:
  sm_token:
    file: ./secrets/sm.token
```

## Health checks

```bash
# CLI ping
php -r '
  require "vendor/autoload.php";
  $c = new \PhpOpcua\SessionManager\Client\ManagedClient();
  exit($c->ping() ? 0 : 1);
'
```

systemd service-level: use `ExecStartPost=` to ping after startup, or rely on `Restart=on-failure`.

## Log routing

The daemon writes via PSR-3 if a logger is wired up programmatically; otherwise it uses `StreamLogger` writing to stderr. For systemd/Docker, stderr is captured by the container/journal — no extra config needed.

To inject a custom logger when starting programmatically:

```php
$daemon = new SessionManagerDaemon(
    socketPath: '/var/run/opcua/sm.sock',
    logger: $monolog,
);
$daemon->run();
```

## Monitoring

The daemon emits PSR-3 logs at key lifecycle points (session open / close / error, command dispatch, cleanup timer, signal handling). Tail with:

```bash
journalctl -u opcua-session-manager -f                   # systemd
docker logs -f opcua-sm                                  # Docker
tail -f /var/log/opcua/sm.stderr.log                     # Supervisor
```

For metrics, wrap `ManagedClient` calls in your application code and emit StatsD / Prometheus from there. The daemon doesn't ship a metrics endpoint by design (keeps the surface tight).

## Upgrades

1. Stop the daemon (`systemctl stop opcua-session-manager`) — all sessions close cleanly via SIGTERM.
2. `composer update php-opcua/opcua-session-manager` in the daemon's deployment.
3. Restart (`systemctl start opcua-session-manager`).

The IPC envelope is versioned but currently stable across v4.x — clients and daemons on different v4.x patches interoperate. Major-version upgrades (v5.x) require coordinated client+daemon redeploy.
