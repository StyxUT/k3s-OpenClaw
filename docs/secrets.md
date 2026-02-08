---
# Secrets

This repo contains Kubernetes manifests for deploying `openclaw-gateway`.

It intentionally includes Secret manifests as *templates* (placeholders) so the
expected keys and wiring are obvious.

Do not commit real secret values.

## Kubernetes Secrets in this repo

This repo commits Secret *templates* as `*.yaml.example`. Real secrets should be
created locally as `*.yaml` (same names, without `.example`) and are ignored by
git via `.gitignore`.

Quickstart:

```bash
cp openclaw-secret.yaml.example openclaw-secret.yaml
cp openclaw-gog-secret.yaml.example openclaw-gog-secret.yaml
cp openclaw-github-secret.yaml.example openclaw-github-secret.yaml
cp openclaw-telegram-secret.yaml.example openclaw-telegram-secret.yaml
```

### `openclaw-secret.yaml.example` (`Secret/openclaw-secrets`)

- Purpose: Gateway authentication token.
- Keys:
  - `gateway_token`: used by the gateway as `OPENCLAW_GATEWAY_TOKEN`.

Where it is used:

- `openclaw-deployment.yaml` sets env var `OPENCLAW_GATEWAY_TOKEN` from this Secret.

### `openclaw-gog-secret.yaml.example` (`Secret/openclaw-gog`)

- Purpose: Provide gogcli OAuth client credentials and headless keyring settings.
- Keys:
  - `credentials.json`: Google OAuth Desktop App client JSON (not a refresh token)
  - `account`: default account email for `gog` (`GOG_ACCOUNT`)
  - `keyring_password`: password for gogcli file keyring (`GOG_KEYRING_PASSWORD`)

Where it is used:

- `openclaw-deployment.yaml`:
  - mounts `credentials.json` to `/home/node/.openclaw/gog/credentials.json`
  - sets env vars `GOG_ACCOUNT`, `GOG_KEYRING_PASSWORD`, `GOG_KEYRING_BACKEND=file`

### `openclaw-github-secret.yaml.example` (`Secret/openclaw-github`)

- Purpose: GitHub token.
- Keys:
  - `token`: used as `GH_TOKEN` / `GITHUB_TOKEN`.

Where it is used:

- `openclaw-deployment.yaml`:
  - initContainer `install-gh` calls GitHub releases API using `GH_TOKEN`.

Note: `install-gog` is pinned and does not require GitHub auth.

### `openclaw-telegram-secret.yaml.example` (`Secret/openclaw-telegram`)

- Purpose: Telegram bot token.
- Keys:
  - `token`: exposed as `TELEGRAM_BOT_TOKEN`.

Where it is used:

- `openclaw-deployment.yaml` sets env var `TELEGRAM_BOT_TOKEN` from this Secret.

## Where secrets are stored at runtime

### In Kubernetes (etcd)

Kubernetes Secrets are stored in etcd (base64-encoded; encryption-at-rest
depends on your cluster configuration).

Authoring in this repo uses `stringData` for readability; the Kubernetes API
stores secrets under `data`.

### On the PVC (`/home/node/.openclaw`)

This deployment mounts a PVC path to `/home/node/.openclaw`.

Some sensitive material is written to this PVC at runtime:

- gogcli refresh tokens and related auth state
  - stored by `gog` in its keyring backend (configured as `file`)
  - location is under `XDG_CONFIG_HOME` / `XDG_DATA_HOME` (set in `openclaw-deployment.yaml`)
  - example config path shown by `gog auth keyring`:
    - `/home/node/.openclaw/xdg-config/gogcli/config.json`

Important implications:

- Deleting the PVC data will delete gogcli tokens and you must re-auth.
- Rotating `keyring_password` without migrating the keyring will orphan existing tokens.

## Recommended operational practices

- Keep real secrets out of this repo; use a private overlay, SOPS/SealedSecrets,
  or a secrets manager.
- Prefer least-privilege scopes for gogcli (e.g., Gmail-only unless needed).
- Restrict cluster access: anyone who can `kubectl exec` into the pod (or read
  the PVC) can access the stored gog token material.

## Namespace and storage notes

- Secrets are namespaced. If you deploy the workload into a non-default namespace
  (for example, `openclaw`), create the Secret resources in that same namespace.
- PersistentVolumes (PVs) are cluster-scoped and bind to exactly one
  PersistentVolumeClaim (PVC).
  - If you move the workload to a new namespace, you cannot reuse an existing PV
    that is already bound to a PVC in another namespace.
  - For NFS-backed storage it is common to create a new static PV that points at
    the same NFS export, and then use `subPath` (as in `openclaw-deployment.yaml`)
    to keep each workload's data in a dedicated subdirectory.

See `docs/storage.md` for a ready-to-fill static NFS PV/PVC example.
