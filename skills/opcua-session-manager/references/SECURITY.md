# Security reference

The daemon holds long-lived OPC UA sessions and the credentials needed to open them. Hardening matters. The package ships with conservative defaults — most production setups need to **tighten** them, not loosen.

## Threat model

The session-manager assumes:

- The host running the daemon is **trusted** — no concurrent untrusted process can read the socket / dump memory
- The IPC socket is **local-only** — never bind to non-loopback addresses (enforced)
- Application code talking to the daemon is **trusted** — it can issue any whitelisted operation
- The OPC UA endpoint is **untrusted** — the server is what we're defending against (use proper TLS / mTLS upstream)

What it explicitly does NOT protect against:

- Root on the host machine (root can read anything)
- A malicious co-tenant on the same machine (mitigate with VM / container isolation)
- A compromised application using the daemon legitimately (audit at the application layer)

## Defense layers

### 1. Local-only IPC

`TcpLoopbackTransport` rejects non-loopback hosts at construction. There is no way to bind `tcp://0.0.0.0:...` or dial a remote daemon's TCP — the loopback check applies to both server bind and client connect.

Unix sockets are intrinsically local (no network exposure). Default mode `0600` restricts to the owning user.

### 2. Socket permission

```bash
--socket=unix:///var/run/opcua/sm.sock --socket-mode=0660
```

Use `0660` + group ownership to grant access to a specific application user without exposing the socket to other users on the host.

### 3. Auth token

Required in production. Three resolution paths, priority order:

1. `OPCUA_AUTH_TOKEN` env var (best for containers — set via `EnvironmentFile=` / Docker secrets)
2. `--auth-token-file=<path>` (best for systemd / bare metal — chmod 600 the file)
3. `--auth-token=<value>` (worst — visible to `ps`; only for non-production)

```php
// Client
new ManagedClient(
    authToken: $_ENV['OPCUA_AUTH_TOKEN']
        ?? @file_get_contents('/etc/opcua/sm.token')
        ?: throw new RuntimeException('No OPC UA auth token configured'),
);
```

Comparison is timing-safe (`hash_equals`). The token is plain bytes — no hashing or HMAC, no nonce. Wrap with TLS / VPN if it has to cross a host (don't — see local-only above).

### 4. Method whitelist

51 allowed operations. The daemon refuses anything else. Setters are blocked (e.g. `setEventDispatcher`, `setTrustStore`, `setRetryDelay`) — they would mutate daemon-side state and bypass the configuration the operator chose at startup.

The whitelist is hardcoded in `CommandHandler` for the built-in methods. Custom modules register their methods through `register()` — those methods are then implicitly whitelisted, but only for the daemon they're loaded into.

### 5. Credential stripping (v4.2.0)

`SessionConfig::SENSITIVE_FIELDS` lists fields stripped from in-memory state after the OPC UA session is up:

- `password`
- `clientKeyPath`
- `caCertPath`
- `userKeyPath`

The daemon needs these during `connect()` to negotiate the secure channel. After that, the active `Client` holds only the negotiated symmetric keys — the original cert/key paths and password are zeroed. A memory dump of the running daemon won't reveal them.

`SessionConfig::sanitized()` returns a logger-safe clone. The CommandHandler always sanitizes before any PSR-3 log call involving the config.

### 6. Cert directory whitelist

```bash
--allowed-cert-dirs=/etc/opcua/certs,/srv/opcua/certs
```

Without this flag, the daemon refuses to load **any** cert file path from an IPC `open` payload — sessions must use auto-generated self-signed certs. With the flag, only paths under (`realpath()`-resolved) listed directories are accepted.

This prevents:

- Path traversal (`/etc/opcua/certs/../../etc/shadow`)
- Symlink attacks (symlink in `/tmp` pointing at a sensitive file)
- Arbitrary file reads via crafted IPC commands

The check uses `realpath()` so symlinks are resolved before the whitelist match.

### 7. Connection limits

| Limit | Default | Behaviour |
| --- | --- | --- |
| `--max-sessions` | 100 | `open` raises `DaemonException` when exceeded |
| Max concurrent socket connections | 50 | Hard-coded daemon-internal limit |
| Connection idle timeout | 30 s | Sockets silent for 30 s are closed |
| Session inactivity timeout (`--timeout`) | 600 s | Cleanup timer removes expired sessions |

Tune `--max-sessions` to your expected concurrent application count. Too low → "max sessions exceeded" errors at peak. Too high → daemon memory pressure.

### 8. Error sanitization

Errors crossing the IPC are sanitized:

- File paths stripped from messages (avoid leaking `/etc/opcua/certs/...`)
- Stack traces truncated (no internal class names exposed)
- Status codes preserved (callers need them for routing)

Raised on the client side as typed exceptions (`DaemonException`, `SessionNotFoundException`, `SerializationException`). The original OPC UA `ServiceException` IS preserved when it's the underlying cause — clients can still match on `StatusCode::isGood()` etc.

### 9. PID lock

The daemon creates a PID file (alongside the socket file by default). A second daemon instance with the same socket exits immediately with a clear "another instance is running" error. Prevents accidental double-start under systemd / Supervisor restart loops.

### 10. Graceful shutdown

SIGTERM and SIGINT are trapped. The daemon:

1. Stops accepting new IPC connections
2. Iterates `SessionStore` and calls `disconnect()` on each session (CloseSession + CloseSecureChannel on the wire — proper UA-level cleanup, no `RST` on the network)
3. Flushes any in-flight AutoPublisher event dispatches
4. Releases the socket file
5. Exits 0

Allow at least 30 seconds for this dance in your supervisor's `TimeoutStopSec` / `stopwaitsecs`.

## Required vs recommended hardening

| Item | Required for prod | Recommended for prod |
| --- | --- | --- |
| Auth token | Yes | Use env var + Docker secrets / systemd EnvironmentFile |
| Local-only IPC | Auto-enforced | n/a |
| Socket mode 0600/0660 | Default 0600 | Tune to match application user |
| `--allowed-cert-dirs` | If accepting cert paths | Always set (even empty list) |
| Method whitelist | Default-on | n/a |
| `--max-sessions` | Default 100 | Tune to actual concurrency |
| Sanitized logs | Default-on | n/a |
| Service-managed (systemd / Supervisor) | n/a | Yes — graceful shutdown matters |
| Read-only mounts for cert dirs | n/a | Mount `--allowed-cert-dirs` `:ro` in Docker |
| Drop privileges | Recommended | Run daemon as a dedicated unprivileged user |

## What's NOT in scope

- **Encryption of the IPC wire**: local-only by design. No TLS for `unix://`, no TLS for `tcp://127.0.0.1`. Use a VPN / SSH tunnel if you absolutely must cross machines (not supported, not recommended).
- **Per-method ACL**: every authenticated caller can invoke any whitelisted method. Granular authorisation belongs in the application layer.
- **Audit logging of every IPC call**: the daemon logs at PSR-3 INFO level for lifecycle events, not per-call. Wire your own listener to PSR-14 events (every call dispatches one) for an audit trail.

## Reporting a vulnerability

The php-opcua org accepts private security advisories via GitHub. Open a private advisory on the affected repo (not a public issue). See the org's `SECURITY` section for the protocol.
