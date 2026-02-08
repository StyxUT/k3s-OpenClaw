# AGENTS.md

This repository is a small Kubernetes manifest bundle for deploying the
`openclaw-gateway` workload.

Contents (today):
- `openclaw-deployment.yaml` (Deployment)
- `openclaw-svc.yaml` (Service)
- `openclaw-secret.yaml.example` (Secret template: gateway token)
- `openclaw-config.yaml` (ConfigMap with `openclaw.json`)
- `openclaw-gog-secret.yaml.example` (Secret template: gog OAuth client + keyring password)
- `openclaw-github-secret.yaml.example` (Secret template: GitHub token)
- `openclaw-telegram-secret.yaml.example` (Secret template: Telegram bot token)
- `docs/gmail-gog.md` (How to configure Gmail via `gog` skill)
- `docs/secrets.md` (How secrets are managed and stored)
- `docs/storage.md` (PV/PVC and NFS binding notes)

Cursor/Copilot rules:
- No `.cursorrules`, `.cursor/rules/`, or `.github/copilot-instructions.md` found.

---

Build / Lint / Test Commands

There is no traditional build/test toolchain in this repo. "Testing" is
validating the manifests (schema + server-side validation) and doing a dry-run
apply.

Prereqs (common):
- `kubectl` configured for your cluster
- Optional linters: `yamllint`, `kubeconform` (or `kubeval`), `kustomize`

Apply (real deployment):
- Apply all manifests in this directory:
  - `kubectl apply -f .`
- Delete resources (be careful):
  - `kubectl delete -f .`

Dry-run / "single test" workflows (recommended):
- Client-side dry run (fast; catches basic YAML/kubectl issues):
  - `kubectl apply --dry-run=client -f openclaw-deployment.yaml`
  - `kubectl apply --dry-run=client -f openclaw-svc.yaml`
  - `kubectl apply --dry-run=client -f openclaw-secret.yaml`
- Server-side dry run (best signal; uses live API validation):
  - `kubectl apply --dry-run=server -f openclaw-deployment.yaml`
- Diff against live cluster (if supported by your kubectl/plugins):
  - `kubectl diff -f .`

Schema validation (acts like unit tests for manifests):
- Validate one file with kubeconform (single-test equivalent):
  - `kubeconform -strict -summary openclaw-deployment.yaml`
- Validate everything in the repo:
  - `kubeconform -strict -summary -ignore-missing-schemas .`

YAML linting:
- Lint a single file:
  - `yamllint openclaw-deployment.yaml`
- Lint the whole directory:
  - `yamllint .`

Observability / smoke checks:
- Watch rollout:
  - `kubectl rollout status deployment/openclaw-gateway`
- Describe resources:
  - `kubectl describe deployment/openclaw-gateway`
  - `kubectl describe svc/openclaw-gateway`
- Inspect pods/logs:
  - `kubectl get pods -l app=openclaw-gateway -o wide`
  - `kubectl logs -l app=openclaw-gateway --tail=200`

If you need to run only one "test":
- Prefer `kubectl apply --dry-run=server -f <file>` (cluster-aware validation).
- If you do not have cluster access, prefer `kubeconform -strict <file>`.

---

Code Style Guidelines (Kubernetes YAML)

General
- Keep files small and single-purpose: one primary resource per file.
- Always include the YAML doc separator `---` at the top of each file.
- Use 2-space indentation; never tabs.
- Keep key ordering conventional: `apiVersion`, `kind`, `metadata`, then `spec`.
- Use stable, explicit values; avoid relying on defaults unless obvious.

Naming
- Resource names: lowercase DNS-1123 labels (e.g., `openclaw-gateway`).
- Labels:
  - Use `app: <name>` consistently for selectors.
  - Keep `metadata.labels` and `spec.selector.matchLabels` aligned.
- Container name should match the app name when there is one container.

Formatting
- Quote strings only when needed (ports and integers as strings when required by
  consuming apps); do not quote booleans or integers unless the field expects a
  string.
- Use double quotes for strings that must remain strings (e.g., env values).
- Prefer lists with `-` on the same indentation level as the list key.
- Avoid trailing whitespace and extra blank lines.

"Imports"
- Not applicable (no language-level imports). If you later add Kustomize/Helm:
  - Prefer Kustomize overlays over copy/paste YAML.
  - Keep bases reusable; keep environment-specific changes in overlays.

Types / API compatibility
- Ensure `apiVersion` matches the target cluster version.
- Avoid deprecated APIs; check with `kubectl api-resources` or release notes.
- Use explicit `containerPort` names when referenced by probes or services.

Deployment conventions
- Always specify `imagePullPolicy` intentionally.
- Prefer pinned images (tag or digest). Avoid `:latest` for production.
- Include readiness and liveness probes when the app exposes a stable port.
- Security contexts:
  - Use least privilege. Add `runAsNonRoot: true` when compatible.
  - Prefer dropping Linux capabilities if the container supports it.
- Storage:
  - Be explicit about `mountPath` and `subPath`.
  - Avoid reusing the same volume name for unrelated purposes unless it is the
    same PVC intentionally.

Service conventions
- Ensure `spec.selector` matches pod labels.
- Prefer `ClusterIP` unless `NodePort` is required.
- Use named ports (`name: http`) and keep service/target ports consistent.

Secrets and config
- Never commit real secrets.
  - `openclaw-secret.yaml` currently contains a literal token value; treat it as
    placeholder/dev-only and replace per environment.
- Prefer `stringData` for authoring; Kubernetes stores base64 in `data`.
- Keep secret keys lowercase with underscores (`gateway_token`).

Error handling (how to change manifests safely)
- Validate every change with at least one of:
  - `kubectl apply --dry-run=client -f <file>`
  - `kubectl apply --dry-run=server -f <file>`
  - `kubeconform -strict <file>`
- When changing selectors/labels, do it as an atomic change across:
  - Deployment pod template labels
  - Deployment selector matchLabels
  - Service selector
- When changing ports, update consistently across:
  - `containerPort`
  - probes (`tcpSocket.port` or HTTP)
  - Service `port`/`targetPort`

Repo-specific notes
- Workload name/labels: `openclaw-gateway` and `app: openclaw-gateway`.
- Exposed port: 18789 (Service `NodePort`, container port).
- Env vars: `OPENCLAW_GATEWAY_TOKEN` comes from Secret `openclaw-secrets`.
- Volume mounts target `/home/node/.openclaw` and `/home/node/openclaw`.

Namespace + PV/PVC
- The core manifests in this repo currently set `metadata.namespace: openclaw`.
- PersistentVolumes (PVs) are cluster-scoped and bind 1:1 to a PVC.
  - If you migrate the workload across namespaces, create a new PV for the new
    PVC (even if it points at the same NFS export); use `subPath` to isolate data.

---

Agent Workflow Expectations

When modifying this repo as an agent:
- Keep changes minimal, reversible, and focused on the requested behavior.
- Do not introduce new tooling/config files unless they clearly improve
  validation/operability (and explain why).
- Do not rotate or generate secrets without explicit instruction.
- Prefer adding a small validation command snippet to this file if you add a new
  tool (e.g., Kustomize/Helm).
