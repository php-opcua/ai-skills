# Security

OPC UA security has three concerns; this bundle wires Symfony ergonomics around all three.

## 1. Transport security (security policy + mode)

`security_policy` selects the cipher suite; `security_mode` selects what's protected.

| Policy | Algorithm | Status |
|---|---|---|
| `None` | — | No protection. Dev only. |
| `Basic128Rsa15` | RSA-1.5 + AES-128-CBC + SHA-1 | **Deprecated** (1.04). Don't use. |
| `Basic256` | RSA-OAEP + AES-256-CBC + SHA-1 | **Deprecated** (1.04). Don't use. |
| `Basic256Sha256` | RSA-OAEP + AES-256-CBC + SHA-256 | **Recommended** default for RSA. |
| `Aes128Sha256RsaOaep` | RSA-OAEP-SHA256 + AES-128-CBC + SHA-256 | Modern RSA alt. |
| `Aes256Sha256RsaPss` | RSA-PSS + AES-256-CBC + SHA-256 | Strongest RSA. |
| `ECC_nistP256` | ECDH-NIST-P-256 + AES-128-GCM + SHA-256 | **Recommended** modern. |
| `ECC_nistP384` | ECDH-NIST-P-384 + AES-256-GCM + SHA-384 | Strongest ECC. |
| `ECC_brainpoolP256r1` | ECDH-Brainpool-P-256 + AES-128-GCM + SHA-256 | EU regulators. |
| `ECC_brainpoolP384r1` | ECDH-Brainpool-P-384 + AES-256-GCM + SHA-384 | EU regulators. |

| Mode | Sign | Encrypt | Use |
|---|:-:|:-:|---|
| `None` | × | × | Dev / Docker test servers |
| `Sign` | ✓ | × | Authenticated, plaintext payload |
| `SignAndEncrypt` | ✓ | ✓ | Default for production |

Production rule: `Basic256Sha256` + `SignAndEncrypt` for RSA, `ECC_nistP256` + `SignAndEncrypt` for ECC.

## 2. Client certificate

When a policy + mode are set and no client cert is supplied, the client auto-generates one in memory at connect time (RSA-2048 for RSA policies, EC matching the curve for ECC). Convenient for dev, useless for production server-side trust.

For production:

```yaml
php_opcua_symfony_opcua:
    connections:
        default:
            client_certificate: '%env(OPCUA_CLIENT_CERT)%'
            client_key: '%env(OPCUA_CLIENT_KEY)%'
            ca_certificate: '%env(OPCUA_CA_CERT)%'
```

```env
OPCUA_CLIENT_CERT=/etc/opcua/client.pem
OPCUA_CLIENT_KEY=/etc/opcua/client.key
OPCUA_CA_CERT=/etc/opcua/ca.pem
```

File permissions:
- `client_certificate` → 0644
- `client_key` → 0600 (readable by web user only)
- `ca_certificate` → 0644

`allowed_cert_dirs` (in `session_manager` config) optionally restricts which directories the daemon may load cert/key files from.

### Generating a client cert

```bash
openssl req -x509 -newkey rsa:2048 -keyout client.key -out client.pem \
    -days 365 -nodes \
    -subj "/CN=symfony-opcua-client" \
    -addext "subjectAltName=URI:urn:symfony-opcua:client,DNS:client.local"
```

The `URI:urn:...` SAN is **required** by spec. Many servers reject certs without it.

## 3. User authentication

### Anonymous

```yaml
connections:
    default:
        username: ~
        password: ~
        user_certificate: ~
```

### Username / Password

```yaml
connections:
    default:
        username: '%env(OPCUA_USERNAME)%'
        password: '%env(OPCUA_PASSWORD)%'
```

The password is encrypted by the client using the server's certificate before transmission (when `security_mode != None`). Never enable `security_mode: None` with username auth.

### X.509 user token

```yaml
connections:
    default:
        user_certificate: '/etc/opcua/user-operator.pem'
        user_key: '/etc/opcua/user-operator.key'
```

User cert ≠ client cert. Client cert authenticates the application, user cert authenticates the operator. Many production servers require both.

## Trust store

`FileTrustStore` persists trusted and rejected server certs on disk:

```yaml
php_opcua_symfony_opcua:
    connections:
        default:
            trust_store_path: '%kernel.project_dir%/var/opcua-trust-store'
            trust_policy: fingerprint
            auto_accept: false
            auto_accept_force: false
```

### Layout on disk

```
var/opcua-trust-store/
├── trusted/
│   └── <fingerprint>.der
└── rejected/
    └── <fingerprint>.der
```

### Trust policies

- `fingerprint` — match by SHA-256 of cert bytes. Survives expiry / renewal as long as key is reused.
- `fingerprint+expiry` — also rejects expired certs.
- `full` — full X.509 chain validation against `ca_certificate`. Most strict.

### TOFU bootstrap

Set `auto_accept: true` for dev:

1. App connects, server presents cert
2. Cert is unknown → added to `trusted/` and accepted
3. Subsequent connects validate against the stored cert

Combine with `opcua-cli trust opc.tcp://server:4840` to seed the store before deploying:

```bash
opcua-cli trust opc.tcp://server:4840 --trust-store=var/opcua-trust-store
```

Then deploy with `auto_accept: false` — any cert change throws `UntrustedCertificateException`.

### Manual cert management

```php
use PhpOpcua\Client\OpcUaClientInterface;

public function __construct(private OpcUaClientInterface $opcua) {}

// Add a specific server cert
$this->opcua->trustCertificate(file_get_contents('/path/to/server.der'));

// Remove by fingerprint
$this->opcua->untrustCertificate('a1:b2:c3:...');
```

## Exception handling

Three exceptions specific to security:

| Exception | When | What to do |
|---|---|---|
| `UntrustedCertificateException` | Server cert not in trust store under `Strict` policy | Inspect cert, trust manually or rotate |
| `WriteTypeDetectionException` | `auto_detect_write_type` failed (couldn't read node metadata) | Disable auto-detect for that node, supply `$type` explicitly |
| `WriteTypeMismatchException` | Detected type doesn't match the PHP value | Convert the value, or supply `$type` explicitly |

```php
use PhpOpcua\Client\Exception\UntrustedCertificateException;
use Psr\Log\LoggerInterface;
use Symfony\Component\HttpKernel\Exception\ServiceUnavailableHttpException;

public function show(LoggerInterface $logger): array
{
    try {
        return ['state' => $this->opcua->read('i=2259')->getValue()];
    } catch (UntrustedCertificateException $e) {
        $logger->warning('OPC UA server cert untrusted', [
            'fingerprint' => $e->getCertificateFingerprint(),
            'subject' => $e->getCertificateSubject(),
            'endpoint' => $e->getEndpointUrl(),
        ]);
        throw new ServiceUnavailableHttpException(message: 'Plant connectivity unavailable');
    }
}
```

## Secrets at rest

Always keep credentials in `.env.local` / `.env.local.php`, never in `config/packages/*.yaml` (which is committed). For multi-tenant setups, use Symfony's `Secret` storage:

```bash
php bin/console secrets:set OPCUA_PASSWORD
```

Then reference as usual:
```yaml
password: '%env(OPCUA_PASSWORD)%'
```

Rotating `APP_SECRET` requires re-encrypting the secret store.

## Network-level

- **Firewall.** Restrict outbound 4840/4843/4848/etc. to OPC UA server hosts.
- **mTLS at the network edge** (HAProxy / Envoy) when crossing untrusted segments. OPC UA's transport security is good; defense-in-depth is cheap.
- **IPC auth token.** Set `OPCUA_AUTH_TOKEN` even on single-host setups.

## Compliance

- **OPC UA Part 2** defines security objectives. The vocabulary (`SecureChannel`, `Session`, `UserTokenPolicy`) comes from there.
- **NIST SP 800-82** treats OPC UA as a "field bus" — ICS/SCADA controls apply.
- **EU NIS2** explicitly covers industrial control systems. Audit trail for cert changes is helpful — listen to `ServerCertificateManuallyTrusted` with a queued handler that writes to an immutable audit table.

## Quick security checklist

- [ ] `security_policy` ≥ `Basic256Sha256` in production
- [ ] `security_mode` = `SignAndEncrypt`
- [ ] Real client cert supplied (not auto-generated)
- [ ] Cert key file mode `0600`, owned by web user
- [ ] `trust_store_path` set to a deploy-volume directory (not tmpfs)
- [ ] `auto_accept` = `false` after bootstrap
- [ ] `OPCUA_AUTH_TOKEN` set, ≥ 32 bytes random
- [ ] `OPCUA_PASSWORD` / X.509 user creds only in `.env.local` or secrets store
- [ ] Daemon socket `socket_mode` = `0600`
- [ ] Firewall restricts outbound OPC UA ports to known servers
