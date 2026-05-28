# Pitfalls reference

Mistakes AI coding assistants frequently make when generating bash that invokes `opcua-cli`. Read before producing CLI snippets.

## 1. Unquoted NodeId strings

**Wrong**:
```bash
opcua-cli read opc.tcp://server:4840 ns=2;s=Temp
# → bash parses ';' as a command separator: runs `opcua-cli read opc.tcp://server:4840 ns=2` then `s=Temp`
```

**Right**:
```bash
opcua-cli read opc.tcp://server:4840 'ns=2;s=Temp'
```

Single quotes preserve the literal string. Double quotes work too if you need `$variable` expansion: `"ns=2;s=$NAME"`.

## 2. Mixing `--debug` with `--json` to stdout

**Wrong**:
```bash
opcua-cli read opc.tcp://server:4840 'i=2259' --json --debug | jq '.value'
# → debug lines on stdout interleave with the JSON output, breaking jq
```

**Right**:
```bash
opcua-cli read opc.tcp://server:4840 'i=2259' --json --debug-stderr | jq '.value'
# debug → stderr, JSON → stdout, jq is happy
```

Or use `--debug-file=PATH` for a separate log:
```bash
opcua-cli read … --json --debug-file=/tmp/opcua.log | jq '.value'
```

## 3. `--json` or `--debug` with `explore`

```bash
opcua-cli explore opc.tcp://server:4840 --json
# → error: "explore does not support --json"
opcua-cli explore opc.tcp://server:4840 --debug
# → error: "explore does not support --debug; use --debug-file=PATH"
```

The TUI takes over the terminal. Both flags would corrupt the display, so they're explicitly rejected. Use `--debug-file=PATH` if you need logs.

## 4. Forgetting Windows is unsupported by `explore`

```bash
# On Windows
opcua-cli explore opc.tcp://server:4840
# → Error: "explore is not yet supported on Windows"
```

The `php-tui/php-tui` library doesn't support Windows terminals (yet). Windows users should use `browse --recursive --depth=N --json | jq`.

## 5. Loops of `opcua-cli read` instead of one PHP script

**Wrong** (N round trips, each ~150 ms):
```bash
for node in ns=2;s=A ns=2;s=B ns=2;s=C ns=2;s=D; do
    opcua-cli read opc.tcp://server:4840 "$node" --json
done
```

**Right** (one round trip via `readMulti`):
```php
$client = ClientBuilder::create()->connect('opc.tcp://server:4840');
$result = $client->readMulti()
    ->node('ns=2;s=A')->value()
    ->node('ns=2;s=B')->value()
    ->node('ns=2;s=C')->value()
    ->node('ns=2;s=D')->value()
    ->execute();
```

The CLI is for ad-hoc / scripting. For >5 ops in a row, use `opcua-client` directly.

## 6. `--password` from a shell history

```bash
opcua-cli read … -u operator -p hunter2
# 'hunter2' goes into ~/.bash_history AND is visible to `ps`
```

**Right** — env var (with `chmod 600` on the source):
```bash
export OPC_PASSWORD=$(cat /run/secrets/opc_pwd)
opcua-cli read … -u operator -p "$OPC_PASSWORD"
```

Or read from stdin (when supported — the CLI currently does not prompt; this is a roadmap item).

## 7. Default trust-store path mismatches across users

```bash
# Run as 'opcua-svc' user — trust goes into /home/opcua-svc/.opcua
opcua-cli trust opc.tcp://server:4840

# Then in another script running as 'cron' user — trust store NOT shared
opcua-cli read opc.tcp://server:4840 'i=2259'  # may fail trust policy check
```

**Right** — explicit `--trust-store=PATH` (e.g. `/etc/opcua/trust` with group-readable permissions):
```bash
opcua-cli trust opc.tcp://server:4840 --trust-store=/etc/opcua/trust
opcua-cli read opc.tcp://server:4840 'i=2259' --trust-store=/etc/opcua/trust
```

## 8. `--type` for auto-detectable writes

Not wrong, but suboptimal — when you know the type, pass it:

```bash
# Slow (read-before-write to detect type)
opcua-cli write opc.tcp://server:4840 'ns=2;i=1001' 42

# Fast (no extra round-trip)
opcua-cli write opc.tcp://server:4840 'ns=2;i=1001' 42 --type=Int32
```

Detect once, hard-code in your script.

## 9. `dump:nodeset` against a large server without `--namespace`

```bash
opcua-cli dump:nodeset opc.tcp://big-plc:4840 --output=dump.xml
# Walks every namespace including ns=0 (the standard OPC UA namespace, ~3000 nodes).
# Can take 5+ minutes and produce a 50 MiB+ XML.
```

**Right** — target your application's namespace:
```bash
opcua-cli dump:nodeset opc.tcp://big-plc:4840 --output=dump.xml --namespace=2
```

To find which namespace you want:
```bash
opcua-cli read opc.tcp://big-plc:4840 'i=2255' --json | jq -r '.value[]'
```
That returns the `NamespaceArray` — the index of your vendor namespace is usually 2 or 3.

## 10. `watch --interval=10`

```bash
opcua-cli watch opc.tcp://server:4840 'ns=2;s=Temp' --interval=10
# Polls 100 times per second. Hammers the server.
```

**Right** — let subscription mode handle it (default):
```bash
opcua-cli watch opc.tcp://server:4840 'ns=2;s=Temp'
# Server pushes changes when the value updates, at the publishingInterval the server picks (usually 500 ms).
```

Polling mode (`--interval`) is for servers that don't support subscriptions (rare).

## 11. Writing the wrong type silently

```bash
opcua-cli write opc.tcp://server:4840 'ns=2;s=Setpoint' 42 --type=Int32
# Server expects Double — write fails with BadTypeMismatch
```

**Right** — match the node's actual DataType:
```bash
# Discover first
opcua-cli read opc.tcp://server:4840 'ns=2;s=Setpoint' --attribute=DataType --json

# Then write with the right type
opcua-cli write opc.tcp://server:4840 'ns=2;s=Setpoint' 42.0 --type=Double
```

Or omit `--type` to let the CLI auto-detect (costs a read-before-write but never mismatches).

## 12. Generated code in `vendor/`

```bash
opcua-cli generate:nodeset Vendor.NodeSet2.xml --output=vendor/my-package/src
# Composer overwrites vendor/ on every install. Your work is gone.
```

**Right** — output into your application's own `src/`:
```bash
opcua-cli generate:nodeset Vendor.NodeSet2.xml \
    --output=src/Generated/Vendor \
    --namespace='App\OpcUa\Vendor'
```

Commit the result to your repo.

## 13. Forgetting `composer dump-autoload` after generating

```bash
opcua-cli generate:nodeset Vendor.NodeSet2.xml --output=src/Generated/Vendor --namespace='App\OpcUa\Vendor'
php my-app.php
# → Error: class App\OpcUa\Vendor\VendorRegistrar not found
```

**Right** — refresh the autoloader after generating into a new namespace:
```bash
composer dump-autoload
```

Or ensure your `composer.json` already maps `App\\OpcUa\\` → `src/Generated/`, and the new files get picked up automatically on next request.

## 14. Hard-killing `explore` and breaking the terminal

```
$ opcua-cli explore opc.tcp://server:4840
# … in another terminal …
$ kill -9 <pid>
# Original terminal is now in raw mode — characters don't echo, cursor invisible
```

`SIGKILL` skips the TUI's cleanup handlers.

**Recovery**:
```bash
reset
# Or:
stty sane
```

Use `q` / `Ctrl-C` instead of `kill -9` to quit cleanly.

## 15. Expecting `opc.https://` to work out of the box

```bash
opcua-cli read opc.https://server:443/UA/TestServer 'i=2259'
# → Error: only opc.tcp:// is supported
```

`opcua-cli` v4.4 only speaks `opc.tcp://`. `opc.https://` requires `php-opcua/opcua-client-ext-transport-https` installed alongside, and currently the CLI doesn't have first-class HTTPS endpoint support — would need either a future CLI release or a custom PHP wrapper using `setTransport(new HttpsTransport(...))`.

## 16. Treating exit code 0 as "the value was good"

Exit code 0 means "the command ran". A successful command can still return a `Bad*` StatusCode on a read.

**Wrong**:
```bash
if opcua-cli read opc.tcp://server:4840 'ns=2;s=Temp' > /tmp/temp; then
    use_it /tmp/temp
fi
```

**Right** — check the StatusCode:
```bash
result=$(opcua-cli read opc.tcp://server:4840 'ns=2;s=Temp' --json) || exit 2

status=$(echo "$result" | jq -r '.statusCode')
if [[ "$status" != "0" ]]; then
    echo "Bad status: $status"
    exit 4
fi

value=$(echo "$result" | jq -r '.value')
use_it "$value"
```

The CLI maps Bad statuses to exit code 4 for `read` / `write`, but the value is still emitted on stdout — you might pipe garbage into the next stage if you don't check.

## 17. Pinning to the wrong `opcua-client` version

`opcua-cli` v4.4 requires `opcua-client` ^4.4 — the constraint is `"php-opcua/opcua-client": "^4.4.0"` in the CLI's composer.json. Mixing major-minor versions across the ecosystem (e.g. CLI v4.4 + opcua-client v4.3) will fail composer resolution.

**Right** — let composer's lock-step resolution do the work; don't pin sub-deps manually.
