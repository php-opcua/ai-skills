# Recipes — complete working examples

Copy-pasteable bash for common operations. Every recipe is end-to-end runnable.

## R1 — Quick probe

```bash
opcua-cli endpoints opc.tcp://server:4840 -t 2
```

Lists every endpoint with security policy and user token policies. Times out after 2 seconds if the server is unreachable.

## R2 — Read the server's State (canonical "is it running?" check)

```bash
state=$(opcua-cli read opc.tcp://server:4840 'i=2259' --json -t 5 | jq -r '.value')
case "$state" in
    0) echo "Running" ;;
    1) echo "Failed" ;;
    2) echo "NoConfiguration" ;;
    3) echo "Suspended" ;;
    4) echo "Shutdown" ;;
    5) echo "Test" ;;
    6) echo "CommunicationFault" ;;
    7) echo "Unknown" ;;
    *) echo "Unrecognized: $state"; exit 1 ;;
esac
```

## R3 — Browse the Objects folder to a CSV

```bash
opcua-cli browse opc.tcp://server:4840 /Objects --recursive --depth=5 --json \
    | jq -r '
        def walk($prefix):
            .references[]
            | "\($prefix)\(.displayName),\(.nodeId),\(.nodeClass)",
              (if .children then .children | walk("\($prefix)  ") else empty end);
        walk("")
    ' \
    > address-space.csv
```

## R4 — Watch a temperature, alert if it crosses a threshold

```bash
opcua-cli watch opc.tcp://server:4840 'ns=2;s=Temperature' --json --debug-stderr \
    | jq --unbuffered -c 'select(.value > 90)' \
    | while IFS= read -r alert; do
          ts=$(jq -r '.timestamp' <<< "$alert")
          v=$(jq -r '.value' <<< "$alert")
          curl -sX POST https://slack-webhook \
               -d "{\"text\":\"⚠️ Temperature $v at $ts\"}"
      done
```

## R5 — Write a setpoint with auto-type-detection

```bash
opcua-cli write opc.tcp://server:4840 'ns=2;s=Setpoint' 42.5
```

The CLI does a read-before-write to discover the DataType, then writes with the right `BuiltinType`. One extra round-trip — fine for one-shot scripts.

For hot paths, hard-code the type:

```bash
opcua-cli write opc.tcp://server:4840 'ns=2;s=Setpoint' 42.5 --type=Double
```

## R6 — Secure connection (Basic256Sha256 + username)

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' \
    -s Basic256Sha256 -m SignAndEncrypt \
    -u operator -p "$OPC_PASSWORD"
```

Or with X.509 client auth:

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' \
    -s Basic256Sha256 -m SignAndEncrypt \
    --cert=/etc/opcua/client.pem \
    --key=/etc/opcua/client.key \
    --ca=/etc/opcua/ca.pem
```

## R7 — TOFU bootstrap then secure connect

```bash
# Step 1 — trust the server's cert (Trust On First Use)
opcua-cli trust opc.tcp://server:4840 --trust-store=/etc/opcua/trust

# Step 2 — every subsequent secure connect validates against the stored thumbprint
opcua-cli read opc.tcp://server:4840 'i=2259' \
    -s Basic256Sha256 -m SignAndEncrypt \
    -u operator -p "$OPC_PASSWORD" \
    --trust-store=/etc/opcua/trust \
    --trust-policy=fingerprint
```

To audit / clean up:

```bash
opcua-cli trust:list --trust-store=/etc/opcua/trust
opcua-cli trust:remove ab:cd:12:34:... --trust-store=/etc/opcua/trust
```

## R8 — Generate PHP types from a vendor NodeSet2

```bash
opcua-cli generate:nodeset /opt/vendor/Vendor.NodeSet2.xml \
    --output=src/Generated/Vendor \
    --namespace='App\OpcUa\Vendor'

composer dump-autoload

# Use in PHP
php -r '
require "vendor/autoload.php";
$client = (new \PhpOpcua\Client\ClientBuilder())
    ->loadGeneratedTypes(new \App\OpcUa\Vendor\VendorRegistrar())
    ->connect("opc.tcp://vendor.example:4840");
var_dump($client->read("ns=2;s=AxisInfo")->getValue());
'
```

## R9 — Dump a server's vendor namespace to NodeSet2.xml

```bash
opcua-cli dump:nodeset opc.tcp://vendor.example:4840 \
    --output=Vendor.NodeSet2.xml \
    --namespace=2 \
    -u operator -p "$OPC_PASSWORD" \
    -s Basic256Sha256 -m SignAndEncrypt
```

Then feed it into `generate:nodeset` (R8) to get typed PHP for that server.

## R10 — CI health gate (GitHub Actions)

```yaml
- name: Probe OPC UA server
  run: |
    opcua-cli endpoints "${{ secrets.OPC_ENDPOINT }}" --json -t 5 > endpoints.json
    if ! jq -e '.endpoints | length > 0' endpoints.json; then
        echo "::error::No endpoints discovered"
        exit 1
    fi

- name: Read server state
  run: |
    state=$(opcua-cli read "${{ secrets.OPC_ENDPOINT }}" 'i=2259' --json -t 5 | jq -r '.value')
    if [[ "$state" != "0" ]]; then
        echo "::error::Server not running (state=$state)"
        exit 1
    fi
```

## R11 — Cron probe with structured log

```bash
#!/usr/bin/env bash
# /usr/local/bin/opcua-probe.sh
# Run via cron: */5 * * * * opcua-svc /usr/local/bin/opcua-probe.sh

set -euo pipefail
LOG=/var/log/opcua-probe.ndjson
ENDPOINT=opc.tcp://plc.example:4840

timestamp=$(date -Iseconds)
if result=$(opcua-cli read "$ENDPOINT" 'i=2259' --json -t 3 --debug-stderr 2>>/var/log/opcua-probe-debug.log); then
    state=$(jq -r '.value' <<< "$result")
    status=$(jq -r '.statusCode' <<< "$result")
    echo "{\"ts\":\"$timestamp\",\"endpoint\":\"$ENDPOINT\",\"state\":$state,\"status\":$status,\"result\":\"ok\"}" >> "$LOG"
else
    rc=$?
    echo "{\"ts\":\"$timestamp\",\"endpoint\":\"$ENDPOINT\",\"result\":\"fail\",\"rc\":$rc}" >> "$LOG"
fi
```

## R12 — Bulk-read many nodes (CSV)

For a one-shot, this is fine — but a PHP script using `readMulti()` is faster (1 round-trip vs N).

```bash
ENDPOINT=opc.tcp://server:4840
NODES=('ns=2;s=Temp' 'ns=2;s=Pressure' 'ns=2;s=Humidity' 'i=2259')

{
    echo "node,value,statusCode,type,sourceTimestamp"
    for node in "${NODES[@]}"; do
        opcua-cli read "$ENDPOINT" "$node" --json -t 3 \
            | jq -r '"\(.node),\(.value),\(.statusCode),\(.type),\(.sourceTimestamp)"'
    done
} > snapshot.csv
```

The PHP equivalent (faster):

```php
$client = ClientBuilder::create()->connect('opc.tcp://server:4840');
$result = $client->readMulti()
    ->node('ns=2;s=Temp')->value()
    ->node('ns=2;s=Pressure')->value()
    ->node('ns=2;s=Humidity')->value()
    ->node('i=2259')->value()
    ->execute();
foreach ($result as $i => $dv) {
    fputcsv(STDOUT, [$nodes[$i], $dv->getValue(), $dv->statusCode, $dv->type?->name]);
}
```

## R13 — Connect to ECC NIST endpoint

```bash
opcua-cli read opc.tcp://server:4848 'i=2259' \
    -s ECC_nistP256 -m SignAndEncrypt \
    -u admin -p admin123
```

The CLI auto-generates an ECC client certificate matching the policy's curve. No additional cert provisioning needed for the auto-accept test server.

## R14 — Pipe `watch` into a database

```bash
opcua-cli watch opc.tcp://server:4840 'ns=2;s=Temperature' --json --debug-stderr \
    | jq -c '{ts:.timestamp, v:.value, q:.statusCode}' \
    | while IFS= read -r row; do
          psql -d telemetry -c "INSERT INTO temp_readings (ts, value, quality) VALUES ('$(jq -r .ts <<<"$row")', $(jq .v <<<"$row"), $(jq .q <<<"$row"))"
      done
```

## R15 — Discover security policies before connecting

```bash
# What does the server advertise?
opcua-cli endpoints opc.tcp://server:4840 --json \
    | jq -r '.endpoints[] | "\(.securityPolicy) \(.securityMode)"' \
    | sort -u
# Output:
# http://opcfoundation.org/UA/SecurityPolicy#Basic256Sha256 Sign
# http://opcfoundation.org/UA/SecurityPolicy#Basic256Sha256 SignAndEncrypt
# http://opcfoundation.org/UA/SecurityPolicy#None None
```

Then pick the strongest your client cert supports:

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' -s Basic256Sha256 -m SignAndEncrypt --cert=... --key=...
```

## R16 — TUI exploration with logs captured to a file

```bash
opcua-cli explore opc.tcp://server:4840 --debug-file=/tmp/explore-debug.log
# In another terminal:
tail -f /tmp/explore-debug.log
```

Useful for verifying browses, refreshes, and any client-side warnings happen as expected while you navigate the TUI.

## R17 — Detect "is this an OPC UA server at all?"

```bash
if opcua-cli endpoints opc.tcp://unknown:4840 --json -t 2 2>/dev/null; then
    echo "Looks like OPC UA"
else
    echo "Not an OPC UA endpoint (rc=$?)"
fi
```

Exit code 2 = connection failure (probably not OPC UA). Exit code 0 = valid GetEndpoints response.

## R18 — Verify a generated PHP class against the server

```bash
# 1. Dump the namespace
opcua-cli dump:nodeset opc.tcp://server:4840 --output=Server.NodeSet2.xml --namespace=2

# 2. Generate PHP
opcua-cli generate:nodeset Server.NodeSet2.xml --output=tmp/Generated --namespace='App\Tmp'

# 3. Diff against current committed code
diff -r src/Generated/ tmp/Generated/ && echo "OK — no drift"
```

Run in CI to catch when a server's NodeSet changes (and your typed PHP is out-of-date).
