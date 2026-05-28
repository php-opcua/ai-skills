# Security reference

Connecting to an OPC UA endpoint with anything beyond plain `None` requires a security policy + (optionally) message-level signing/encryption + authentication. The CLI exposes every knob the underlying `opcua-client` does.

## Quick recipes

### Open (no security)

```bash
opcua-cli read opc.tcp://localhost:4840 'i=2259'
# Implicit: -s None -m None, anonymous user token
```

### Username / password over a secure channel

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' \
    -s Basic256Sha256 -m SignAndEncrypt \
    -u operator -p "$OPC_PASSWORD"
```

### X.509 client certificate auth

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' \
    -s Basic256Sha256 -m SignAndEncrypt \
    --cert=/etc/opcua/client.pem \
    --key=/etc/opcua/client.key \
    --ca=/etc/opcua/ca.pem
```

### ECC connection

```bash
opcua-cli read opc.tcp://server:4848 'i=2259' \
    -s ECC_nistP256 -m SignAndEncrypt \
    -u admin -p admin123
```

## Security policy choice

10 policies supported (matching `opcua-client`'s enum):

| Policy | Notes |
| --- | --- |
| `None` | No security. Cleartext over the wire. Development only. Implied default when `-s` omitted. |
| `Basic128Rsa15` | Deprecated by spec but still common on older servers. |
| `Basic256` | Deprecated. |
| `Basic256Sha256` | Modern RSA default. Pick this for new RSA deployments. |
| `Aes128Sha256RsaOaep` | RSA, newer suite. |
| `Aes256Sha256RsaPss` | Strongest RSA. |
| `ECC_nistP256` | NIST P-256 curve (elliptic curve). |
| `ECC_nistP384` | NIST P-384 curve. |
| `ECC_brainpoolP256r1` | Brainpool P-256r1. |
| `ECC_brainpoolP384r1` | Brainpool P-384r1. |

Server-side support varies. **Always run `endpoints <url>` first** to see what's advertised:

```bash
opcua-cli endpoints opc.tcp://server:4840 --json | jq -r '.endpoints[].securityPolicy' | sort -u
```

## Security mode

`-m` / `--security-mode`:

| Mode | Wire-level guarantees |
| --- | --- |
| `None` | Cleartext. Requires `-s None`. |
| `Sign` | Message-level signatures (integrity, no confidentiality). |
| `SignAndEncrypt` | Signatures + AES encryption. Production default. |

A mismatch like `-s Basic256Sha256 -m None` is rejected by the server with `BadSecurityModeRejected`.

## Client certificate

For any non-None policy, the CLI sends a `CreateSession.ClientCertificate`. Two flavours:

### Auto-generated (default — works for any policy)

If you DON'T pass `--cert` / `--key`, the CLI generates a self-signed certificate in memory per session. The matching curve/keysize is picked from the security policy (RSA 2048 for RSA policies, P-256 / P-384 for ECC).

The server may still reject it (untrusted). For `uanetstandard-test-suite` auto-accept servers it works as-is. For production, the server admin must trust your cert first.

### Bring your own

```bash
opcua-cli ... \
    --cert=/etc/opcua/client.pem \
    --key=/etc/opcua/client.key \
    --ca=/etc/opcua/ca.pem
```

- `--cert` — PEM-encoded leaf certificate
- `--key` — PEM-encoded private key
- `--ca` — (optional) PEM-encoded CA certificate; the CLI appends it to build a chain

The application URI in the cert (`X509v3 Subject Alternative Name → URI:urn:...`) must match what the server's user token policy expects. Most servers accept anything that looks like `urn:*`.

## User authentication

Three modes, mutually exclusive:

### Anonymous (default)

```bash
opcua-cli read opc.tcp://server:4840 'i=2259'
# No -u, no --cert → server picks the Anonymous user token policy
```

### Username / password

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' -u operator -p s3cret
```

`-u` / `--username` and `-p` / `--password`. The password is shipped in the `UserIdentityToken`. When `-s` is non-None, the password is encrypted by the server's identity-token policy.

### X.509 user identity token

Set the cert as the **user** identity (different from the client app cert):

There is currently no separate `--user-cert` flag at the CLI surface (the underlying `setUserCertificate(...)` exists in `opcua-client`). For now, use `-u` / `-p` or wire a small PHP script if you need X.509 user auth — `opcua-cli` v4.4 does not expose it directly.

## Trust store (TOFU)

The CLI ships a per-user trust store at `$HOME/.opcua` by default. When you connect to a new server with non-None security, the server's certificate is captured (TOFU — Trust On First Use). Future connections to the same endpoint validate against the stored thumbprint.

### TOFU bootstrap

```bash
opcua-cli trust opc.tcp://server:4840                         # capture + trust
opcua-cli trust:list                                          # see what's trusted
opcua-cli trust:remove ab:cd:12:34:...                        # untrust
```

### Trust policies

`--trust-policy=POLICY`:

| Policy | Behaviour |
| --- | --- |
| `fingerprint` | Match the stored SHA-256 thumbprint. **Default.** |
| `fingerprint+expiry` | Match thumbprint AND check the cert's `notAfter`. |
| `full` | Walk the CA chain back to a trusted root. |
| (omitted via `--no-trust-policy`) | Accept any cert (TOFU disabled). Dev only. |

```bash
opcua-cli read opc.tcp://server:4840 'i=2259' \
    -s Basic256Sha256 -m SignAndEncrypt \
    -u operator -p "$OPC_PASSWORD" \
    --trust-policy=fingerprint+expiry
```

The trust store path can be overridden with `--trust-store=PATH` (e.g. `/etc/opcua/trust` for a system-wide store managed by Ops).

## Combinations you'll hit in practice

| Server / situation | Flags |
| --- | --- |
| Local test server, anonymous | (defaults) |
| `uanetstandard-test-suite` userpass server (port 4841) | `-s Basic256Sha256 -m SignAndEncrypt -u admin -p admin123` |
| `uanetstandard-test-suite` certificate server (4842) | `-s Basic256Sha256 -m SignAndEncrypt --cert=… --key=… --ca=…` |
| `uanetstandard-test-suite` ECC NIST (4848) | `-s ECC_nistP256 -m SignAndEncrypt -u admin -p admin123` |
| `uanetstandard-test-suite` HTTPS Binary (4852) | (CLI does not yet have `opc.https://` first-class support; needs `opcua-client-ext-transport-https` installed alongside — would require adding `--endpoint-url` parsing of `opc.https://`) |
| Production with trust store | `-s Basic256Sha256 -m SignAndEncrypt -u … -p … --trust-policy=fingerprint+expiry --trust-store=/etc/opcua/trust` |

## Password handling

- **Never** hard-code passwords in shell history. Either:
  ```bash
  # From an environment variable
  opcua-cli read … -p "$OPC_PASSWORD"
  
  # From a file (chmod 600)
  opcua-cli read … -p "$(cat /run/secrets/opc_pwd)"
  ```
- **Beware of `ps`** — `--password=secret` shows up in process lists. The env-var / file pattern avoids it (the shell substitutes before `opcua-cli` is exec'd, but the resolved value still appears in `/proc/<pid>/cmdline`). For maximum safety, write a small PHP script via `opcua-client` that reads the password from stdin or a secret manager.

## Timeouts

`-t SECONDS` / `--timeout=SECONDS` — default 5 s. Applies to connect, send, receive. For:

- **CI gates / health probes**: `-t 2` (fail fast)
- **High-latency networks / VPN**: `-t 30`
- **Long-running `dump:nodeset`**: the timeout is per-call, not total. Set to ~15 s and let large dumps make many calls.

## Endpoints discovery without security

`endpoints` does pure discovery (no session). It works regardless of `-s` / `-m`:

```bash
opcua-cli endpoints opc.tcp://server:4840 -t 2
```

Recommended pattern: probe first, then pick the right `-s` / `-m` / `-u` / cert combination based on what's advertised.
