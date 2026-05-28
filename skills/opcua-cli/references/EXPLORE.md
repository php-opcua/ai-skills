# `explore` — interactive TUI reference

A full-screen terminal UI for browsing an OPC UA server's address space. Backed by `php-tui/php-tui` ^0.2.1.

## Platform support

| OS | Supported |
| --- | --- |
| Linux | yes |
| macOS | yes |
| Windows | **No.** Prints a clear "not yet supported" error — upstream `php-tui/php-tui` does not yet support Windows terminals. |

For Windows users: use `browse --recursive --json | jq` for ad-hoc exploration.

## Usage

```bash
opcua-cli explore opc.tcp://localhost:4840
opcua-cli explore opc.tcp://localhost:4840 -u operator -p operator123
opcua-cli explore opc.tcp://localhost:4840 -s Basic256Sha256 -m SignAndEncrypt --cert=… --key=…
```

All [security options](SECURITY.md) work normally.

## Layout

```
┌─ Tree ───────────────────────┬─ Details ──────────────────────────────────┐
│ ▾ Root (i=84)                │ NodeId:        ns=0;i=85                   │
│   ▾ Objects (i=85)           │ BrowseName:    Objects                     │
│     ▸ Server (i=2253)        │ DisplayName:   Objects                     │
│     ▾ MyPLC (ns=2;i=1000)    │ NodeClass:     Object                      │
│       ─ Temp (ns=2;s=…) =22.5│ Description:   The browse entry point …    │
│       ─ Press(ns=2;s=…) =1.2 │                                            │
│   ▸ Types (i=86)             │ References (12):                           │
│   ▸ Views (i=87)             │   forward — Organizes → Server             │
│                              │   forward — Organizes → MyPLC              │
│                              │   …                                        │
├─ Log ─────────────────────────────────────────────────────────────────────┤
│ 10:30:45 INFO  Browsed ns=2;i=1000 — 14 references                        │
│ 10:30:46 INFO  Refreshed ns=2;s=Temp — value=22.5                         │
└───────────────────────────────────────────────────────────────────────────┘
```

Three panes:

- **Tree** (left, ~40 % width) — collapsible tree view of the address space
- **Details** (right) — the focused node's attributes and references
- **Log** (bottom) — diagnostic messages, last N lines visible

The active pane has a highlighted border. Tab switches focus.

## Keys

### Global

| Key | Action |
| --- | --- |
| `q` / `Esc` | Quit |
| `Tab` | Switch focused pane (Tree → Details → Log → Tree) |
| `?` | Show help overlay (where supported) |

### Tree pane

| Key | Action |
| --- | --- |
| `↑` / `k` | Move selection up |
| `↓` / `j` | Move selection down |
| `→` / `Enter` | Expand (Object/View/Type) or open (Variable shows in Details) |
| `←` / `h` | Collapse current node; if already collapsed, jump to parent |
| `Home` | Jump to root |
| `End` | Jump to last visible node |
| `r` | Refresh the selected Variable's Value attribute |
| `PgUp` / `PgDn` | Page up/down |

### Details pane

| Key | Action |
| --- | --- |
| `↑` / `↓` | Scroll the references list |
| `r` | Refresh the focused node's attributes |

### Log pane

| Key | Action |
| --- | --- |
| `↑` / `↓` | Scroll the log |
| `End` | Jump to newest entry |
| `c` | Clear |

## Output redirection

`explore` takes over the terminal. To capture debug logs without disrupting the display:

```bash
# Wrong — corrupts the TUI
opcua-cli explore opc.tcp://localhost:4840 --debug

# Right — debug to stderr (terminal-safe? no, also corrupts)
# Right — debug to a file
opcua-cli explore opc.tcp://localhost:4840 --debug-file=/tmp/opcua-explore.log
```

While `explore` is running, in another terminal:

```bash
tail -f /tmp/opcua-explore.log
```

The TUI explicitly **rejects** `--json` (output would be eaten by the screen) and `--debug` (writes to stdout, corrupts the display). Both cause an immediate error before the TUI starts.

## What it does NOT do

- **No write / call.** Read-only browser. To set a value, exit and use `opcua-cli write`.
- **No subscriptions.** Variable values shown in Details are fetched on-demand (`r` to refresh). No live updates.
- **No history.** Use `opcua-cli` history commands (when added) or a PHP script via `opcua-client`.
- **No multi-server tabs.** One server per `explore` invocation.

## When the connection drops

The TUI shows a banner: "Disconnected — press `r` to reconnect or `q` to quit". The address space tree is preserved (cached); only refreshes block until reconnect.

## Performance

- Initial connect: ~150 ms (one CreateSession + ActivateSession).
- Browse of a node: ~50 ms per server round-trip (cached after first browse).
- Refresh of a Variable's Value: ~30 ms per round-trip.

Large servers (10k+ nodes) are fine — the tree is lazy: children are fetched on expand, never all at once.

## Debugging the TUI itself

If the TUI behaves strangely (garbled output, stuck cursor):

1. `Ctrl-C` or `q` to quit cleanly (sends SIGINT, restores terminal state).
2. If terminal is broken: `reset` (a Unix command that re-initializes the terminal).
3. Capture verbose logs: `--debug-file=/tmp/explore.log` and inspect.

The `php-tui/php-tui` library handles terminal restoration via atexit/SIGINT handlers. A hard kill (`SIGKILL`) leaves the terminal in raw mode — `reset` fixes it.

## Alternatives for non-TTY environments

For automation / CI / SSH-without-TTY, use `browse` instead:

```bash
opcua-cli browse opc.tcp://server:4840 --recursive --depth=10 --json > address-space.json
jq '.references | length' address-space.json
```

`explore` requires a real interactive terminal (checks `php_sapi_name() === 'cli'` and `stream_isatty(STDIN)`). It refuses to run if stdin isn't a TTY.
