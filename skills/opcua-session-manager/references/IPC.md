# IPC reference

How the client and daemon talk to each other. The v4.2.0 IPC layer (`src/Ipc/`) abstracts Unix-domain sockets and TCP loopback behind a single `TransportInterface`; `WireMessageCodec` handles NDJSON framing of typed envelopes.

## Transport selection

`TransportFactory::defaultEndpoint()` picks per OS:

- **Linux / macOS** → `unix:///tmp/opcua-session-manager.sock`
- **Windows** → `tcp://127.0.0.1:<port>` (port assigned per user)

Override via constructor or CLI:

```php
new ManagedClient(endpoint: 'unix:///var/run/opcua/sm.sock');
new ManagedClient(endpoint: 'tcp://127.0.0.1:9876');
```

```bash
php bin/opcua-session-manager --socket=unix:///var/run/opcua/sm.sock
php bin/opcua-session-manager --socket=tcp://127.0.0.1:9876
```

The endpoint URI scheme determines the transport class. A scheme-less path (`/tmp/foo.sock`) is treated as `unix://`.

## Loopback-only guard

`TcpLoopbackTransport` checks the host portion of the URI at construction. Any value other than `127.0.0.1`, `::1`, or `localhost` raises `InvalidArgumentException` immediately. This applies to BOTH the daemon-side `SocketServer` listen and the client-side `Connect`. There is no way to bind / dial a non-loopback TCP — the daemon is intentionally local-only.

To cross machines, use SSH port-forwarding or a VPN — never expose the daemon's TCP socket to the network.

## Wire format

Each message is a JSON envelope, line-framed (NDJSON):

```
{"v":1,"id":"<uuid>","cmd":"open","args":{...}}\n
{"v":1,"id":"<uuid>","cmd":"invoke","args":{"session":"...","method":"read","params":[...]}}\n
{"v":1,"id":"<uuid>","result":{...}}\n
{"v":1,"id":"<uuid>","error":{"type":"DaemonException","message":"...","code":...}}\n
```

`WireMessageCodec` enforces:

- **16 MiB max frame size** — a single envelope larger than 16 MiB raises `SerializationException` (frames are NDJSON-line-delimited, so this caps any single command's payload)
- **32-level max nesting depth** — gadget chains via deeply nested arrays / objects are rejected
- **Binary mode** on the stream — no charset assumptions, raw bytes

If you fork the codec, keep these limits — they exist as belt-and-suspenders against malicious payloads even though the auth-token gate already restricts access.

## Authentication

Three sources, evaluated in priority order:

1. `OPCUA_AUTH_TOKEN` environment variable
2. `--auth-token-file=<path>` — daemon reads file at startup
3. `--auth-token=<string>` — CLI argument (visible to `ps`; use only for non-production)

Client-side:

```php
new ManagedClient(
    endpoint: 'unix:///var/run/opcua/sm.sock',
    authToken: $_ENV['OPCUA_AUTH_TOKEN'] ?? file_get_contents('/etc/opcua/sm.token'),
);
```

Token comparison uses `hash_equals()` — timing-safe, no leak via comparison length.

When no auth-token is set on the daemon (the default), the daemon accepts any connection — only the socket permission bits (`0600` default) protect it. **Always set an auth-token in production**, even with `0600`.

## Connection limits

Daemon-side guards:

| Limit | Default | Behaviour |
| --- | --- | --- |
| `--max-sessions` | 100 | Further `open` commands raise `DaemonException` with code = "max sessions exceeded" |
| Max concurrent socket connections | 50 (hardcoded) | New connections wait or are dropped |
| Connection idle timeout | 30 s | Sockets that don't send any framed message within 30 s are closed |
| Single-request max size | 1 MiB | `--max-request-size` exists for tuning; defaults are conservative |
| Per-session inactivity timeout | `--timeout` (default 600 s) | Cleaned up on the cleanup timer cycle |

## Socket permissions (Unix only)

```bash
php bin/opcua-session-manager --socket=unix:///var/run/opcua/sm.sock --socket-mode=0660
```

The daemon `chmod`s the socket file to the requested mode after `bind()`. To grant access to a specific app user:

```bash
# socket at 0660 owned by opcua:opcua-clients group
sudo chgrp opcua-clients /var/run/opcua/sm.sock
sudo usermod -a -G opcua-clients www-data
```

For local-only on the same user, default `0600` is correct.

## Cert directory whitelist

Sessions can include client / CA cert file paths in the `open` payload (for OPC UA security). To prevent path traversal / arbitrary file reads, restrict allowed directories:

```bash
php bin/opcua-session-manager --allowed-cert-dirs=/etc/opcua/certs,/srv/opcua/certs
```

Without `--allowed-cert-dirs`, no cert paths are accepted at all — sessions must use the auto-generated self-signed cert path.

Paths are checked via `realpath()` before any file I/O. Symlinks pointing outside the whitelist are rejected.

## Custom transports

You can write your own by implementing `TransportInterface`. Useful for:

- TLS-protected TCP for cross-host setups (not supported out-of-the-box; explicitly rejected by `TcpLoopbackTransport`)
- Named pipes on Windows (today's default uses TCP loopback)
- Pure-process testing (in-memory transport)

```php
namespace App\OpcUa\Ipc;

use PhpOpcua\SessionManager\Ipc\TransportInterface;
use PhpOpcua\SessionManager\Ipc\AbstractStreamTransport;

final class TlsTcpTransport extends AbstractStreamTransport
{
    public function __construct(/* TLS context, host, port */) { /* ... */ }

    protected function openStream(): mixed
    {
        // open a TLS stream, return resource
    }

    // AbstractStreamTransport handles NDJSON framing once the stream is open
}
```

Register via `TransportFactory::register('tls', fn ($uri) => new TlsTcpTransport(...))` on both client and daemon side, then use `tls://...` URIs.

## Idiomatic client setup across OSes

For an application that runs on Linux, macOS, AND Windows:

```php
$endpoint = $_ENV['OPCUA_SOCKET']                                              // explicit override
    ?? PhpOpcua\SessionManager\Ipc\TransportFactory::defaultEndpoint();         // OS-aware default

$client = new ManagedClient(
    endpoint: $endpoint,
    authToken: $_ENV['OPCUA_AUTH_TOKEN'] ?? null,
);
```

`TransportFactory::defaultEndpoint()` returns the appropriate URI for the current `PHP_OS_FAMILY`. The daemon's CLI defaults follow the same logic.
