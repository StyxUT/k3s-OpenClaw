---
# Browser in Pod (Headless Chromium)

This deployment can run a headless Chromium sidecar inside the same Kubernetes
pod as the OpenClaw gateway.

Why this approach:

- No dependency on a workstation browser or extension relay.
- OpenClaw connects to a local CDP endpoint (`127.0.0.1`) inside the pod.
- Browser state (profile) can be persisted via the same PVC.

## How it works

- The Deployment adds a `chromium` sidecar (`chromedp/headless-shell`).
- The sidecar exposes the Chrome DevTools Protocol (CDP) on port `9222`.
- The OpenClaw config sets a browser profile pointing at `http://127.0.0.1:9222`.

## Kubernetes notes

- `/dev/shm`: Chrome is sensitive to small shared memory in containers.
  This repo mounts an in-memory `emptyDir` at `/dev/shm` for the sidecar.
- Readiness probe: the CDP sidecar only listens on `127.0.0.1` by default, so the
  sidecar readiness probe uses an `exec` probe against `127.0.0.1:9222`.
- Persistence: the browser profile uses `--user-data-dir=/data` and `/data` is
  backed by the existing PVC (`openclaw/data/chromium/user-data`).
- Security: this sidecar runs with `--no-sandbox`. If your cluster policies allow
  the Chromium sandbox, consider removing `--no-sandbox` and validating it still
  boots reliably.

## Tool access (agent)

The agent only sees tools that are both:

- present in the gateway tool registry, and
- allowed by `tools.allow` (allowlist), and
- not blocked by `tools.deny`.

Note: denying `group:web` can also block the `browser` tool (depending on how your
OpenClaw build groups tools). If the agent claims it cannot use `browser` while
`openclaw browser ...` commands work, remove `group:web` from `tools.deny`.

## Quick smoke test

After deploying, you can check CDP from inside the pod:

```bash
kubectl -n openclaw exec deploy/openclaw-gateway -c chromium -- \
  wget -qO- http://127.0.0.1:9222/json/version
```

And verify OpenClaw can see the browser:

```bash
kubectl -n openclaw exec deploy/openclaw-gateway -c openclaw-gateway -- \
  node dist/index.js browser status --browser-profile pod
```
