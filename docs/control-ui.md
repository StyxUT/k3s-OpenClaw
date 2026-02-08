---
# Control UI (Web) Pairing

When you open the OpenClaw Control UI from a *new browser profile* (or after
clearing site data), it is treated as a new device.

If `gateway.controlUi.allowInsecureAuth` is `false` (recommended), the gateway
requires a one-time device pairing approval.

Typical symptom in the browser:

- Disconnected `1008` (pairing required)

## Approve the new browser device

List pending pairing requests:

```bash
kubectl -n openclaw exec deploy/openclaw-gateway -c openclaw-gateway -- \
  node dist/index.js devices list --json
```

Approve by `requestId`:

```bash
kubectl -n openclaw exec deploy/openclaw-gateway -c openclaw-gateway -- \
  node dist/index.js devices approve <requestId>
```

### One-liner: approve the first pending request

This pulls the first pending `requestId` and approves it.
Requires `jq` on the machine running `kubectl`.

```bash
REQ_ID="$(kubectl -n openclaw exec deploy/openclaw-gateway -c openclaw-gateway -- \
  node dist/index.js devices list --json | jq -r '.pending[0].requestId // empty')" && \
  test -n "$REQ_ID" && \
  kubectl -n openclaw exec deploy/openclaw-gateway -c openclaw-gateway -- \
    node dist/index.js devices approve "$REQ_ID"
```

After approving, reload the Control UI tab; it should connect and remain paired.
