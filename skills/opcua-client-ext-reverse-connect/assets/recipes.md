# Recipes — complete working examples

Copy-pasteable snippets for OPC UA Reverse Connect. Every recipe is end-to-end runnable. Once a recipe yields a `Client`, use the ordinary `opcua-client` API.

## R1 — Accept one reverse connection, read a value (simplest)

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Client\ExtReverseConnect\ReverseConnectClientFactory;
use PhpOpcua\Client\ExtReverseConnect\ReverseConnectListener;
use PhpOpcua\Client\ExtReverseConnect\ReverseHelloValidator;
use PhpOpcua\Client\Security\SecurityMode;
use PhpOpcua\Client\Security\SecurityPolicy;

$validator = new ReverseHelloValidator(['urn:opcua:testserver:nodes']);
$listener  = new ReverseConnectListener('0.0.0.0', 4840, $validator);
$listener->listen();

try {
    $session = $listener->accept(timeoutSeconds: 30.0);

    $client = (new ReverseConnectClientFactory())->buildClient(
        $session,
        static fn (ClientBuilder $b) => $b
            ->setSecurityPolicy(SecurityPolicy::None)
            ->setSecurityMode(SecurityMode::None),
    );

    try {
        echo "Server: {$session->serverUri}\n";
        echo 'Value: ' . $client->read('ns=2;s=Sensors/Temp')->getValue() . "\n";
    } finally {
        $client->disconnect();
    }
} finally {
    $listener->close();
}
```

## R2 — Ephemeral port + discover the bound port

Useful when something else (a test harness, an out-of-band channel, a `StartReverseConnect` method call) tells the server where to dial.

```php
$listener = new ReverseConnectListener('0.0.0.0', 0, $validator);  // 0 = kernel-assigned
$listener->listen();
[, $port] = explode(':', $listener->getBindAddress());
$port = (int) $port;

echo "Listening on port {$port}; tell the server to reverse-connect here.\n";
$session = $listener->accept(timeoutSeconds: 30.0);
```

## R3 — Long-running multi-server loop

One listener, many servers, resilient to rejected/garbage frames.

```php
$listener = new ReverseConnectListener('0.0.0.0', 4840, $validator);
$listener->listen();
$factory  = new ReverseConnectClientFactory();

while (true) {
    try {
        $session = $listener->accept(timeoutSeconds: 5.0);
    } catch (\PhpOpcua\Client\ExtReverseConnect\Exception\ReverseConnectTimeoutException) {
        continue;                                   // idle tick — re-arm
    } catch (
        \PhpOpcua\Client\ExtReverseConnect\Exception\ReverseConnectRejectedException |
        \PhpOpcua\Client\ExtReverseConnect\Exception\ReverseHelloParseException $e
    ) {
        fwrite(STDERR, "Refused RHE: {$e->getMessage()}\n");  // socket already closed
        continue;
    }

    $client = $factory->buildClient(
        $session,
        static fn ($b) => $b
            ->setSecurityPolicy(SecurityPolicy::None)
            ->setSecurityMode(SecurityMode::None),
    );

    try {
        $state = $client->read('i=2259')->getValue();          // ServerStatus.State
        echo "{$session->serverUri} state={$state}\n";
    } finally {
        $client->disconnect();
    }
}
```

## R4 — Secure reverse connect (Sign & Encrypt)

Production shape: the whitelist gates the announced URI, the secure channel + certificate trust prove identity.

```php
use PhpOpcua\Client\Security\SecurityMode;
use PhpOpcua\Client\Security\SecurityPolicy;

$client = (new ReverseConnectClientFactory())->buildClient(
    $session,
    static function (\PhpOpcua\Client\ClientBuilder $b): void {
        $b->setSecurityPolicy(SecurityPolicy::Basic256Sha256)
          ->setSecurityMode(SecurityMode::SignAndEncrypt)
          ->setClientCertificate('/etc/pki/client-cert.pem', '/etc/pki/client-key.pem')
          ->setTrustedServerCertificates('/etc/pki/trusted');
        // identity tokens, logger, dispatcher — anything ClientBuilder supports
    },
    readTimeout: 10.0,
);
```

> The listener bind interface (`0.0.0.0` vs `127.0.0.1`) does not affect certificate validation — that's driven by the announced `endpointUrl` and the server certificate. See `references/SECURITY.md`.

## R5 — Observability via PSR-14 events

```php
use PhpOpcua\Client\ExtReverseConnect\Event\ReverseConnectAccepted;
use PhpOpcua\Client\ExtReverseConnect\Event\ReverseConnectRejected;
use PhpOpcua\Client\ExtReverseConnect\Event\ReverseHelloReceived;

$dispatcher = new MyPsr14Dispatcher();   // your PSR-14 EventDispatcherInterface
$dispatcher->listen(ReverseHelloReceived::class, fn ($e) =>
    $log->info('RHE received', ['serverUri' => $e->message->serverUri]));
$dispatcher->listen(ReverseConnectAccepted::class, fn ($e) =>
    $metrics->increment('reverse_connect.accepted'));
$dispatcher->listen(ReverseConnectRejected::class, fn ($e) =>
    $log->warning('RHE rejected', ['serverUri' => $e->message->serverUri, 'reason' => $e->reason]));

$listener = new ReverseConnectListener(
    '0.0.0.0', 4840, $validator,
    logger: $psr3Logger,
    dispatcher: $dispatcher,
);
```

Events are constructed only when a dispatcher is supplied — zero overhead otherwise.

## R6 — Retry the trigger connect through a fresh server's boot window

When you also drive the server (e.g. calling `StartReverseConnect` over a normal session), a just-booted server may answer `BadServerHalted` (`0x800E0000`) until it's Running.

```php
use PhpOpcua\Client\ClientBuilder;
use PhpOpcua\Client\Exception\ConnectionException;
use PhpOpcua\Client\Exception\ServiceException;
use PhpOpcua\Client\Security\SecurityMode;
use PhpOpcua\Client\Security\SecurityPolicy;

function connectWithReadinessRetry(string $endpoint, float $budget = 15.0): \PhpOpcua\Client\Client
{
    $deadline = microtime(true) + $budget;
    $last = null;
    do {
        try {
            return ClientBuilder::create()
                ->setSecurityPolicy(SecurityPolicy::None)
                ->setSecurityMode(SecurityMode::None)
                ->connect($endpoint);
        } catch (ServiceException | ConnectionException $e) {
            $last = $e;
            usleep(250_000);
        }
    } while (microtime(true) < $deadline);

    throw $last;
}
```

## R7 — Inspect or branch on a message without throwing

```php
use PhpOpcua\Client\ExtReverseConnect\ReverseHelloParser;
use PhpOpcua\Client\ExtReverseConnect\ReverseHelloValidator;

$message = ReverseHelloParser::parse($rawFrameBytes);     // ReverseHelloMessage
$validator = new ReverseHelloValidator(['urn:a', 'urn:b']);

if ($validator->isAccepted($message)) {                   // bool, no exception
    echo "trusted: {$message->serverUri} @ {$message->endpointUrl}\n";
} else {
    echo "untrusted: {$message->serverUri}\n";
}
```
