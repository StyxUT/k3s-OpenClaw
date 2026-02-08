---
# Gmail Access via gog Skill (Kubernetes)

This repo deploys `openclaw-gateway` on Kubernetes and enables Gmail access
using the bundled `gog` skill (gogcli).

This document describes:

- which manifests and settings enable Gmail
- where secrets live (Kubernetes Secrets vs on-disk token store)
- how to run fully headless OAuth inside the pod

## Components

- `openclaw-config.yaml`: ConfigMap containing `openclaw.json`
- `openclaw-deployment.yaml`: Deployment for `openclaw-gateway`
- `openclaw-gog-secret.yaml.example`: Secret template containing gog OAuth client JSON and keyring password

## How the gog skill is enabled

`openclaw-config.yaml` sets:

- `skills.allowBundled: ["gog"]`
- `skills.entries.gog.enabled: true`

`openclaw-deployment.yaml` mounts the ConfigMap into the container at:

- `/bootstrap/openclaw.json` (initContainer)

An initContainer (`seed-openclaw-config`) copies `/bootstrap/openclaw.json` to the
PVC-backed config path (`/home/node/.openclaw/openclaw.json`) if the file does not
already exist.

Another initContainer (`enforce-openclaw-security`) then enforces certain
security-critical settings on every start using `node dist/index.js config set ...`
with `OPENCLAW_CONFIG_PATH=/home/node/.openclaw/openclaw.json`.

## Installing the gog binary

`openclaw-deployment.yaml` includes an initContainer `install-gog` which:

- downloads a pinned gogcli release (`0.9.0` by default)
- writes the `gog` binary to the PVC-backed directory `/home/node/.openclaw/bin/gog`

The main container then has `PATH` set to include `/home/node/.openclaw/bin`.

## Secrets and where they are stored

### Kubernetes Secrets (authoring)

Authoring happens via `stringData` in these manifests:

- `openclaw-secret.yaml.example` (`openclaw-secrets`): `gateway_token`
- `openclaw-github-secret.yaml.example` (`openclaw-github`): GitHub token (optional; used by `install-gh`)
- `openclaw-telegram-secret.yaml.example` (`openclaw-telegram`): Telegram bot token (optional)
- `openclaw-gog-secret.yaml.example` (`openclaw-gog`):
  - `credentials.json`: Google OAuth desktop client JSON (NOT a refresh token)
  - `account`: default Google account for gog
  - `keyring_password`: encryption password for gogcli file-keyring

Kubernetes stores these values base64-encoded in etcd.

### OAuth refresh token storage (runtime)

The refresh token is NOT stored in a Kubernetes Secret.
It is created during `gog auth add ...` and stored by gogcli in its keyring.

This deployment forces a headless-friendly keyring:

- `GOG_KEYRING_BACKEND=file`
- `GOG_KEYRING_PASSWORD` from the `openclaw-gog` Secret

On disk, gogcli writes its config under:

- `XDG_CONFIG_HOME=/home/node/.openclaw/xdg-config`
- `~/.openclaw/xdg-config/gogcli/config.json`

And stores encrypted keyring entries under its data directory (XDG).
Because `/home/node/.openclaw` is PVC-backed, tokens persist across pod restarts.

## Headless OAuth inside the pod

Prereqs:

- you have created a Google Cloud OAuth Client (Desktop)
- you copied `openclaw-gog-secret.yaml.example` to `openclaw-gog-secret.yaml` and pasted the OAuth client JSON into `credentials.json`

Apply manifests:

```bash
kubectl apply -f openclaw-config.yaml
kubectl apply -f openclaw-gog-secret.yaml
kubectl apply -f openclaw-deployment.yaml
kubectl rollout restart deployment/openclaw-gateway
kubectl rollout status deployment/openclaw-gateway
```

Note: if you're deploying into the `openclaw` namespace, either ensure the
manifests include `metadata.namespace: openclaw` (as this repo does for the core
workload) or add `-n openclaw` to the commands.

Run the auth flow:

```bash
POD="$(kubectl get pods -l app=openclaw-gateway --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"

kubectl exec -it "$POD" -c openclaw-gateway -- sh -lc 'gog auth keyring'
kubectl exec -it "$POD" -c openclaw-gateway -- sh -lc 'gog auth credentials "$GOG_HOME/credentials.json"'
kubectl exec -it "$POD" -c openclaw-gateway -- sh -lc 'gog auth add "$GOG_ACCOUNT" --services gmail --manual --force-consent'
kubectl exec -it "$POD" -c openclaw-gateway -- sh -lc 'gog auth list --check'
```

The `--manual` flow prints a URL; open it on your local machine, approve access,
then paste the code back into the pod prompt.

Verify Gmail access:

```bash
kubectl exec -it "$POD" -c openclaw-gateway -- sh -lc 'gog gmail labels list --json'
```

## Operational notes

- Rotating the OAuth client JSON requires updating the `openclaw-gog` Secret and re-running `gog auth add ...`.
- Rotating `keyring_password` without migrating the keyring will orphan existing encrypted tokens.
- If you delete the PVC path `openclaw/data/.openclaw`, you will delete the stored refresh token and must re-auth.
