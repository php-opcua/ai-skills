---
name: opcua-cli
description: Interact with OPC UA servers from the terminal using php-opcua/opcua-cli. 11 single-shot commands — browse, read, write, watch values in real time, explore the address space with an interactive TUI, discover endpoints, manage server certificate trust, generate typed PHP classes from a NodeSet2.xml, dump a server's address space to NodeSet2.xml. Pipe-friendly JSON output, no framework dependencies. Use this skill whenever the user wants to script OPC UA from a shell, debug a server interactively, generate PHP from a vendor's NodeSet2.xml, or set up CI checks against an OPC UA endpoint.
license: MIT
compatibility: Requires PHP >= 8.2 and ext-openssl. The `explore` interactive TUI requires `php-tui/php-tui` ^0.2.1 and works on Linux/macOS only — Windows is not yet supported by the upstream TUI library. Lock-step with php-opcua/opcua-client v4.4+.
metadata:
  package: php-opcua/opcua-cli
  version: v4.4.0
  ecosystem: php-opcua
---

# php-opcua/opcua-cli — v4.4.0 skill

A pure-PHP, no-framework CLI tool for OPC UA. Wraps `php-opcua/opcua-client` into 11 composable single-shot commands. Every command supports `--json` for piping into `jq` / Unix tools, every connection method (auth, security, cert paths) is the same across commands, and the `explore` command opens a full-screen TUI for ad-hoc address-space browsing.

## When to use this skill

Activate when the user wants to:

- **Probe an OPC UA server** from a shell (`opcua-cli endpoints opc.tcp://server:4840`)
- **Read / write / watch process variables** without writing PHP code
- **Browse / explore** the address space interactively (TUI) or scriptably (JSON)
- **Generate typed PHP classes** from a vendor's NodeSet2.xml (`generate:nodeset`)
- **Export a server's address space** to NodeSet2.xml (`dump:nodeset`)
- **Set up CI integration tests** that read a known node and assert a value
- **Manage the per-user trust store** (`trust`, `trust:list`, `trust:remove`)
- **Pipe OPC UA data** into `jq`, `awk`, Telegraf, monitoring scripts

Do NOT activate for: building a long-running OPC UA service (use `opcua-client` or `opcua-session-manager` directly), library-level integration (use `opcua-client`), or framework-bound integration (use `laravel-opcua` / `symfony-opcua`).

## The 60-second mental model

```
$ opcua-cli <command> <endpoint> [<nodeId> | <args>] [--security-policy=...] [--username=...] [--json] [--debug-*]
                │
                ▼
        Application (src/Application.php) — registers 11 commands, parses argv via ArgvParser
                │
                ▼
        Command (one of 11 in src/Commands/) — implements CommandInterface
                │
                ▼
        CommandRunner (src/CommandRunner.php) — builds the Client (security, trust, timeout)
                │
                ▼
        php-opcua/opcua-client Client.connect() → service call → typed result
                │
                ▼
        OutputInterface (ConsoleOutput tree/text  |  JsonOutput machine-readable)
                │
                ▼
        stdout (data) | stderr (diagnostics) → caller's pipeline
```

Three things to know:

1. **No framework.** Not Symfony Console. Argument parsing is a custom `ArgvParser`, command dispatch is a custom `CommandRunner`. The only runtime dep beyond `opcua-client` is `php-tui/php-tui` (only loaded if you run `explore`).
2. **Two output modes** for every command: human-readable (`ConsoleOutput`, default — colored tree / table) and machine-readable (`--json` → `JsonOutput`). The JSON shape is stable and documented per-command; use it for scripts.
3. **Stdout for data, stderr for diagnostics.** `--debug-stderr` routes log lines to stderr so piping the data output through `jq` stays clean. The TUI command (`explore`) takes over the whole terminal and rejects `--json` / `--debug` (they'd corrupt the display).

## Quick start (90% of use cases fit this shape)

```bash
# Probe — what does this endpoint speak?
opcua-cli endpoints opc.tcp://server.example:4840

# Read the canonical "is the server alive?" node
opcua-cli read opc.tcp://server.example:4840 'i=2259'         # Server.ServerStatus.State

# Read with JSON output → pipe to jq
opcua-cli read opc.tcp://server.example:4840 'ns=2;s=Temp' --json | jq -r '.value'

# Browse one level
opcua-cli browse opc.tcp://server.example:4840 'i=85'           # Objects folder

# Browse a tree, 3 levels deep
opcua-cli browse opc.tcp://server.example:4840 /Objects --recursive --depth=3

# Open the interactive TUI (Linux/macOS only)
opcua-cli explore opc.tcp://server.example:4840

# Write a value
opcua-cli write opc.tcp://server.example:4840 'ns=2;s=Setpoint' 42.5 --type=Double

# Watch a value change in real time
opcua-cli watch opc.tcp://server.example:4840 'ns=2;s=Temp'    # subscription mode (default)
```

## When to load deeper references

| If the task involves… | Read |
| --- | --- |
| Exact flags / JSON output shape for any of the 11 commands | [`references/COMMANDS.md`](references/COMMANDS.md) |
| Security policy choice, certificates, mTLS, username/password, trust store | [`references/SECURITY.md`](references/SECURITY.md) |
| The `explore` interactive TUI — keys, layout, debug-routing | [`references/EXPLORE.md`](references/EXPLORE.md) |
| `generate:nodeset` / `dump:nodeset` — code generation flows | [`references/CODEGEN.md`](references/CODEGEN.md) |
| Pipe-friendly scripting, exit codes, JSON parsing, CI patterns | [`references/SCRIPTING.md`](references/SCRIPTING.md) |
| How the internals are wired (Application, ArgvParser, CommandRunner, Commands) | [`references/ARCHITECTURE.md`](references/ARCHITECTURE.md) |
| Debugging an unfamiliar error or generating wrong shell quoting | [`references/PITFALLS.md`](references/PITFALLS.md) |
| Complete bash one-liners for common ops (cron probe, jq pipelines, CI gate) | [`assets/recipes.md`](assets/recipes.md) |

## The 11 commands

| Command | Description | Connects? |
| --- | --- | --- |
| `browse <endpoint> [<nodeId>]` | List references from a node (default `i=85` Objects). Supports `--recursive`, `--depth=N`. | Yes |
| `read <endpoint> <nodeId>` | Read an attribute (default Value). `--attribute=Value\|DisplayName\|BrowseName\|DataType\|NodeClass\|Description\|AccessLevel\|NodeId`. | Yes |
| `write <endpoint> <nodeId> <value>` | Write a value. `--type=<BuiltinType>` to skip auto-detect. | Yes |
| `watch <endpoint> <nodeId>` | Stream value changes. Subscription mode by default; `--interval=N` switches to polling every N ms. | Yes |
| `explore <endpoint>` | Full-screen TUI. ↑/↓ navigate, →/Enter expand, ← collapse/parent, `r` refresh, `q`/Esc quit. Linux/macOS only. | Yes |
| `endpoints <endpoint>` | Discover server endpoints + their security policies / user token policies. | Yes (discovery — no session) |
| `generate:nodeset <file.NodeSet2.xml>` | Emit PHP enums / DTOs / codecs / registrar / NodeId constants from a NodeSet2.xml. `--output=PATH --namespace=NS`. | **No** |
| `dump:nodeset <endpoint>` | Read a server's address space and export as NodeSet2.xml. `--output=FILE --namespace=N`. | Yes |
| `trust <endpoint>` | Connect, capture the server's certificate, store it in the trust store (TOFU). `--trust-store=PATH --trust-policy=fingerprint\|fingerprint+expiry\|full`. | Yes |
| `trust:list` | List trusted certificates. `--trust-store=PATH`. | **No** |
| `trust:remove <thumbprint>` | Remove a trusted certificate. `--trust-store=PATH`. | **No** |

## Security options (every connection-aware command)

| Option | Short | Values |
| --- | --- | --- |
| `--security-policy=POLICY` | `-s` | `None`, `Basic128Rsa15`, `Basic256`, `Basic256Sha256`, `Aes128Sha256RsaOaep`, `Aes256Sha256RsaPss`, `ECC_nistP256`, `ECC_nistP384`, `ECC_brainpoolP256r1`, `ECC_brainpoolP384r1` |
| `--security-mode=MODE` | `-m` | `None`, `Sign`, `SignAndEncrypt` |
| `--cert=PATH` | | Client certificate (PEM) |
| `--key=PATH` | | Client private key (PEM) |
| `--ca=PATH` | | CA certificate (PEM) |
| `--username=USER` | `-u` | Username for username/password auth |
| `--password=PASS` | `-p` | Password (avoid on shared hosts — visible to `ps`) |
| `--timeout=SECONDS` | `-t` | Connection timeout (default 5 s) |

## Global options (every command)

| Option | Short | Description |
| --- | --- | --- |
| `--json` | `-j` | Machine-readable JSON output (rejected by `explore`) |
| `--debug` | `-d` | Debug log to stdout (rejected by `explore` and incompatible with `--json` piping) |
| `--debug-stderr` | | Debug log to stderr (safe with `--json` piping) |
| `--debug-file=PATH` | | Debug log to a file |
| `--help` | `-h` | Show help (overall or per command: `opcua-cli read --help`) |
| `--version` | `-v` | Show CLI version |

## v4.4.0 alignment

Lock-step with `php-opcua/opcua-client` v4.4.0. The CLI binds to the core directly — no service layer in between — so every server-facing command benefits transparently from the v4.4 additions:

- New `aggregate` / `historyAggregate` / `historyInsert*` / `historyUpdate*` / `historyDelete*` core methods are reachable from CLI flows that already touch history (but the CLI does NOT yet have first-class `history` / `aggregate` subcommands — would be a v4.5+ candidate)
- The HTTPS / Reverse Connect ext transports plug in via `ClientBuilder::setTransport()` — invisible to the CLI surface, just works against `opc.https://` endpoints once those packages are installed alongside
- Pre-generated nodeset types (`php-opcua/opcua-client-nodeset`) are loaded transparently when present (CLI auto-discovers registrars via the autoloader)

## Idiomatic patterns AI agents should follow

1. **Always quote NodeId strings** in shell — `ns=2;s=Temp` has a `;` that bash treats as a command separator. Use single quotes: `'ns=2;s=Temp'`.

2. **`-j` / `--json` for any output you pipe** to `jq` / `awk` / `grep`. Default human output has colors, padding, and tree-drawing characters that break parsing.

3. **`--debug-stderr` instead of `--debug`** when piping. `--debug` writes to stdout and corrupts JSON.

4. **`--type=<BuiltinType>` on writes when you know the type** — saves a read-before-write round trip. Without it, the CLI does an auto-detect read first.

5. **Use the `read` command with `--attribute=DataType`** to discover a node's type before writing. Cheap, no commit.

6. **`trust` before connecting securely to a new server** — `opcua-cli trust opc.tcp://server:4840` pulls the cert into the user trust store, then subsequent commands honour `--trust-policy`. Without trust setup, `--trust-policy=fingerprint` connections to that server fail.

7. **`explore` for ad-hoc, scripts for repeated operations.** The TUI is great for "what does this server have?", terrible for automation. For automation use `browse --recursive --depth=N --json` and parse with `jq`.

8. **`--timeout=2` for fast-fail probes in CI.** The default 5 seconds is fine interactively but stretches CI gates.

9. **Don't use `--password` from a shell.** Prefer reading from a file: `--password="$(cat /run/secrets/opcua_pwd)"` (or use a config file in `php-opcua/laravel-opcua` / `symfony-opcua` integrations).

10. **Exit codes matter**:
    - `0` — success
    - `1` — generic error (parse, validation, business logic)
    - `2` — connection error (DNS, TCP, TLS, OPC UA handshake)
    - `3` — authentication / security error
    - `4` — service-level OPC UA error (`StatusCode != Good`)
    
    Bash scripts can branch on these for selective retry.

## Common pitfalls (read before generating code)

Don't write code that:

- Unquoted NodeIds in shell: `ns=2;s=Temp` — bash parses `;` as a command separator
- `--debug` mixed with `--json` to stdout — debug lines corrupt the JSON output
- `explore` with `--json` or `--debug` — explicitly rejected (would corrupt the TUI)
- Hard-codes `~/.opcua` as trust-store path — different per-user; use `$HOME/.opcua` or pass `--trust-store=PATH` explicitly
- `dump:nodeset` against a huge server without `--namespace=N` filter — the export can run for minutes and produce >100 MiB
- `generate:nodeset` writing into `vendor/` — generated code belongs in your application's `src/`
- Loops calling `opcua-cli read` 100 times — each call pays the connect handshake (~150 ms). Use `watch` or a real PHP script via `opcua-client`.
- `watch --interval=10` (polling every 10 ms) — hammers the server. Stick with subscription mode unless you have a reason.

Full catalog in [`references/PITFALLS.md`](references/PITFALLS.md).

## Related packages in the php-opcua ecosystem

- **`opcua-client`** — the OPC UA client library this CLI wraps. v4.4+ required.
- **`opcua-client-nodeset`** — pre-generated types for 51 OPC Foundation companion specs. Auto-discovered by the CLI (no extra config) when installed in the project.
- **`opcua-session-manager`** — daemon for cross-request session persistence. Not used by `opcua-cli` (each command is one-shot — no benefit). Use it only from long-lived applications.
- **`laravel-opcua`** / **`symfony-opcua`** — framework integrations. The CLI is a separate utility; framework integrations replace it for in-app code, but the CLI stays useful for ops / debugging / generation.
- **`uanetstandard-test-suite`** — Docker-based OPC UA test servers. Useful target for trying every CLI command locally.

## Distribution forms

The CLI ships in three forms (all from the same source):

- **Composer dep**: `composer require php-opcua/opcua-cli` → `vendor/bin/opcua-cli`
- **Global composer install**: `composer global require php-opcua/opcua-cli` → `~/.composer/vendor/bin/opcua-cli` (add to `PATH`)
- **PHAR / standalone binary**: `box.json.dist` builds a single-file PHAR via `bin/opcua-cli`. Distributed via the GitHub Releases workflow (`release-binaries.yml`) — download a PHP-bundled binary for Linux / macOS / Windows.

Recommend `vendor/bin` for project-scoped use; global or PHAR for ops / CI-runner installs.
