# Temporary Node TLS protection

The October 2026 eMudhra compromise requires a Node trust update independently of browser updates.
The Node stages in these Dockerfiles preload `scripts/node-trust.mjs` before package installation and runtime startup.
The local setup/arbiter/merchant/facilitator/client commands also preload it.
The policy uses supported Node CA APIs, requiring Node 22.19+ or Node 24+.

Until Node releases its intermediate-key blocklist, the policy removes the four root public keys in the [incident inventory](https://emudhra.we-have-a-problem.com/roots.json) from the process's selected default CAs.
It preserves other system/default/extra CAs and retains certificate and hostname verification.
The broader root distrust may block legitimate services chaining through those roots; do not disable verification to work around a failure.
OpenSSL CA mode requires explicit `SSL_CERT_FILE` and `SSL_CERT_DIR` paths so the selected trust store can be enumerated without restoring bundled roots.
Explicit custom CA configurations and non-Node clients need their own checks.

Before a local package install, protect the package manager itself:

```sh
ARBITER_TLS_PRELOAD="$(pwd)/scripts/node-trust.mjs"
NODE_OPTIONS="--import=\"$ARBITER_TLS_PRELOAD\" ${NODE_OPTIONS-}" pnpm install
```

Run the opt-in, unauthenticated HTTPS/fetch probe with:

```sh
npm run test:tls-trust
```

All twelve published affected TLS hosts must fail with certificate-validation errors, while the three healthy controls connect.
DNS failures and timeouts fail the check.
The probe is not run during application startup, does not load `.env` files, and does not make payment/inference calls.

This source change does not patch an existing deployed image.
Rebuild, test real outbound integrations and obtain the required rollout authorization.
The Ollama model-download stage in the optional TEE `Dockerfile` is a separate non-Node path; its binary/trust policy has not been verified here.
Prefer the Node-only `Dockerfile.arbiter` and `Dockerfile.merchant` until that path is independently verified.
Remove this temporary policy only after selecting an official fixed Node release and verifying the actual HTTPS/fetch paths again.

Sources: [Node fix](https://github.com/nodejs/node/pull/66871), [Node CA API](https://nodejs.org/api/tls.html#tlssetdefaultcacertificatescerts), [incident test and patch information](https://emudhra.we-have-a-problem.com/).
