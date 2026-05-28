# Scripting reference

Patterns for using `opcua-cli` in bash scripts, CI pipelines, monitoring agents, and Unix data pipelines.

## Output channels

| Stream | Content |
| --- | --- |
| stdout | Data (the answer) — human text by default, JSON with `--json` |
| stderr | Diagnostics — when `--debug-stderr` is set; otherwise stderr is silent on success |
| File | Logs — when `--debug-file=PATH` is set |

**Default behaviour**: a successful command writes only the answer to stdout; on error, prints to stderr AND exits non-zero. No mixing.

## Exit codes

| Code | Meaning | Recover by |
| --- | --- | --- |
| `0` | Success | — |
| `1` | Generic error (parse, validation, business logic) | Fix the input |
| `2` | Connection error (DNS, TCP, TLS, OPC UA handshake) | Retry / check network |
| `3` | Authentication / security error | Check credentials / cert / trust |
| `4` | Service-level OPC UA error (`StatusCode != Good`) | Inspect the StatusCode |

Useful for selective retry:

```bash
opcua-cli read opc.tcp://server:4840 'i=2259'
case $? in
    0) echo "OK" ;;
    2) echo "Retry the network" ;;
    4) echo "Server returned a Bad status" ;;
    *) echo "Hard fail — alert" ;;
esac
```

## JSON output + `jq`

Every command supports `--json`. Output is single-document JSON for one-shot commands; NDJSON (one object per line) for streaming (`watch --json`).

```bash
# Single value from a read
opcua-cli read opc.tcp://server:4840 'i=2259' --json | jq -r '.value'

# All endpoints' security policies
opcua-cli endpoints opc.tcp://server:4840 --json | jq -r '.endpoints[].securityPolicy' | sort -u

# Recursive browse, count nodes
opcua-cli browse opc.tcp://server:4840 /Objects --recursive --depth=10 --json \
    | jq '[.. | objects | select(has("nodeId"))] | length'

# Stream watch into a Telegraf input
opcua-cli watch opc.tcp://server:4840 'ns=2;s=Temp' --json --debug-stderr \
    | jq -c '{measurement: "temp", value: .value, ts: .timestamp}'
```

## Pipe-friendly patterns

### CI health gate

```bash
#!/bin/bash
# fail the build if the server's State isn't Running (= 0)
set -euo pipefail

state=$(opcua-cli read opc.tcp://server:4840 'i=2259' --json -t 5 | jq -r '.value')
if [[ "$state" != "0" ]]; then
    echo "::error::Server state $state (expected 0 = Running)"
    exit 1
fi
echo "✓ Server is running"
```

### Monitor a value crossing a threshold

```bash
opcua-cli watch opc.tcp://server:4840 'ns=2;s=Temp' --json --debug-stderr \
    | jq --unbuffered -c 'select(.value > 90)' \
    | while IFS= read -r line; do
          ts=$(echo "$line" | jq -r '.timestamp')
          v=$(echo "$line" | jq -r '.value')
          echo "[$ts] Temperature too high: $v"
          # send alert
      done
```

### Cron probe (every 5 min)

```bash
#!/bin/bash
# /etc/cron.d/opcua-probe
# */5 * * * * opcua-user /usr/local/bin/opcua-probe.sh

LOG=/var/log/opcua-probe.log
ENDPOINT=opc.tcp://plc.example:4840

if opcua-cli read "$ENDPOINT" 'i=2259' --json -t 2 -d >> "$LOG" 2>&1; then
    : # ok
else
    rc=$?
    echo "[$(date -Iseconds)] PROBE FAILED rc=$rc" >> "$LOG"
    systemd-cat -t opcua-probe -p err <<< "Probe failed against $ENDPOINT (rc=$rc)"
fi
```

### Batch read into CSV

```bash
# Read many nodes, output as CSV
for node in ns=2;s=Temp ns=2;s=Pressure ns=2;s=Humidity; do
    value=$(opcua-cli read opc.tcp://server:4840 "$node" --json -t 2 | jq -r '.value')
    echo "$node,$value"
done > snapshot.csv
```

Better: one PHP script using `opcua-client`'s `readMulti()` — one round-trip instead of N. The CLI is one-shot per call.

### Generate code in CI

```yaml
# .github/workflows/regenerate-types.yml
- name: Regenerate types from vendor NodeSet2
  run: |
    opcua-cli generate:nodeset vendor-files/Vendor.NodeSet2.xml \
        --output=src/Generated/Vendor \
        --namespace='App\OpcUa\Vendor'

- name: Check if regeneration changed anything
  run: |
    if ! git diff --exit-code src/Generated/Vendor; then
        echo "::error::Generated code drifted. Run 'opcua-cli generate:nodeset …' and commit."
        exit 1
    fi
```

## Quoting NodeIds

NodeIds contain `;` and sometimes `/` — bash treats `;` as a command separator. **Always quote**:

```bash
# Wrong — bash parses 'ns=2' as a command, ';s=Temp' as a syntax error
opcua-cli read opc.tcp://server:4840 ns=2;s=Temp

# Right — single quotes preserve the literal string
opcua-cli read opc.tcp://server:4840 'ns=2;s=Temp'

# Also right — escape if you have to use double quotes for $-expansion
opcua-cli read opc.tcp://server:4840 "ns=2;s=$SENSOR_NAME"
```

For path-style NodeIds: `/Objects/MyPLC/Temp` is fine unquoted (no special chars), but make it a habit to quote anyway.

## Long-running commands

`watch`, `explore`, `dump:nodeset` are long-lived. They:

- Trap SIGINT (Ctrl-C) and shutdown cleanly (close subscription, disconnect, write final XML)
- Trap SIGTERM (kill) similarly
- Survive SIGHUP (terminal close) only if launched with `nohup` / `setsid` / inside `tmux` / `screen`

For unattended `watch` on a server, use systemd:

```ini
[Unit]
Description=OPC UA temperature watcher
After=network-online.target

[Service]
Type=simple
User=opcua
ExecStart=/usr/local/bin/opcua-cli watch opc.tcp://plc.example:4840 'ns=2;s=Temp' --json --debug-stderr -u operator -p "${OPC_PASSWORD}"
EnvironmentFile=/etc/opcua/watch.env
Restart=always
RestartSec=10
StandardOutput=append:/var/log/opcua/temp.ndjson
StandardError=journal

[Install]
WantedBy=multi-user.target
```

## Per-command JSON output shape (cheat sheet)

| Command | Shape |
| --- | --- |
| `endpoints` | `{ endpoints: [{ endpointUrl, securityPolicy, securityMode, userIdentityTokens: [...], … }] }` |
| `browse` | `{ node, references: [{ nodeId, browseName, displayName, nodeClass, isForward }] }` (recursive adds `children`) |
| `read` | `{ node, attribute, value, statusCode, type, sourceTimestamp, serverTimestamp }` |
| `write` | `{ node, value, type, statusCode, result }` |
| `watch` | NDJSON: `{ timestamp, value, type, statusCode, sourceTimestamp }` per change |
| `trust` | `{ endpoint, thumbprint, addedAt }` |
| `trust:list` | `{ trustStore, certificates: [{ thumbprint, subject, issuer, notAfter, endpointUrls }] }` |
| `dump:nodeset` | (no JSON — emits XML to `--output`) |
| `generate:nodeset` | (no JSON — emits PHP files to `--output`) |
| `explore` | (no JSON — TUI; rejects `--json`) |

## Performance gotchas

- **Each CLI invocation is one shot** — connect, do work, disconnect. Connect takes ~150 ms. For >5 reads from the same server, write a PHP script using `opcua-client` directly.
- **`watch` in subscription mode** is the right choice for change detection. `--interval=N` polling is for servers that don't support subscriptions (rare — every modern OPC UA server supports them).
- **`dump:nodeset` without `--namespace`** can take many minutes on big servers. Always filter when you can.
- **`generate:nodeset` is CPU-bound, not I/O-bound.** Large NodeSets (10k+ types) take a few seconds. No connection needed.

## Environment variables the CLI cares about

The CLI itself doesn't read env vars (every input is a CLI flag), but the surrounding shell may:

- `HOME` — used for the default trust store path (`$HOME/.opcua`)
- `LANG` / `LC_ALL` — output encoding (some terminals need UTF-8 set)
- `NO_COLOR` — disables ANSI colors in `ConsoleOutput` (respected if set to any non-empty value)
- `TERM` — used by `explore` to pick capabilities (256-color, mouse, etc.)
