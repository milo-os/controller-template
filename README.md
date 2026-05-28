# controller-template

A template repository for building Kubernetes controllers that connect to the Milo control plane. Fork this to bootstrap a new kubebuilder v4 controller service.

## What This Is

This template provides the standard project layout, build tooling, and deployment configuration used by Milo controller services. It includes a working example `Resource` CRD with a reconciler, webhook scaffolding, and a full kustomize deployment tree.

When you fork this template, you replace the `example.miloapis.com` API group, `Resource` kind, and `controller-template` name with your service's specifics.

## How It Connects to the Milo Control Plane

The operator config supports a `kubeconfigPath` field that points the controller at Milo's API server:

```yaml
apiVersion: apiserver.config.miloapis.com/v1alpha1
kind: ControllerTemplateOperator
metricsServer:
  bindAddress: "0"
kubeconfigPath: /etc/milo/kubeconfig
```

When `kubeconfigPath` is empty, the controller uses in-cluster config, which works for local development against a kind cluster that has your CRDs installed.

## Forking This Template

### Prerequisites

| Tool | Minimum version | Install |
|------|----------------|---------|
| Go | 1.25+ | https://go.dev/dl |
| Docker | any recent | https://docs.docker.com/get-docker |
| kind | v0.20+ | `go install sigs.k8s.io/kind@latest` |
| Task runner | v3+ | https://taskfile.dev/installation |
| pnpm | v9+ | `npm install -g pnpm` |
| kubebuilder | v4 (reference only) | https://book.kubebuilder.io/quick-start |

### Automated rename

After forking, run the rename script once to replace all template placeholders:

```bash
chmod +x hack/rename.sh
./hack/rename.sh \
  --service-name billing \
  --api-group billing.miloapis.com \
  --kind BillingAccount
```

Use `--dry-run` to preview changes without writing anything:

```bash
./hack/rename.sh \
  --service-name billing \
  --api-group billing.miloapis.com \
  --kind BillingAccount \
  --dry-run
```

### What gets renamed

| Placeholder | Replaced with |
|-------------|---------------|
| `controller-template` | `--service-name` value (e.g. `billing`) |
| `example.miloapis.com` | `--api-group` value (e.g. `billing.miloapis.com`) |
| `Resource` (CamelCase kind) | `--kind` value (e.g. `BillingAccount`) |
| `resource` (lowercase kind) | lowercase of `--kind` (e.g. `billingaccount`) |
| `ControllerTemplateOperator` | `<Kind>Operator` (e.g. `BillingAccountOperator`) |
| `CONTROLLER_TEMPLATE_API_` | `<SERVICE>_API_` env prefix (e.g. `BILLING_API_`) |
| `go.miloapis.com/controller-template` | `go.miloapis.com/<service-name>` |

File and directory names containing these placeholders are also renamed (e.g. `cmd/controller-template/` and `resource_types.go`).

### Verify nothing was missed

```bash
grep -r "controller-template\|example\.miloapis\.com" \
  --include="*.go" --include="*.yaml" --include="*.ts" --include="*.tsx" .
```

An empty result means all placeholders were replaced. Any hits in `zz_generated.*` files are expected — they will be overwritten by the next step.

### After renaming

```bash
task generate && task manifests   # regenerate deepcopy, CRD, RBAC, and webhook manifests
task build && task test           # confirm it compiles and tests pass
git add -A && git commit -m "rename: controller-template -> your-service-name"
```

### Troubleshooting

**Webhook cert not ready**
The controller pod starts before cert-manager has issued the webhook certificate. Wait ~30 s and check:
```bash
kubectl -n controller-system get certificate
kubectl -n controller-system describe validatingwebhookconfiguration
```
If the certificate is stuck, confirm cert-manager is installed: `kubectl get pods -n cert-manager`.

**Image pull errors**
The dev overlay references a locally-built image that hasn't been pushed to the kind registry yet. Run:
```bash
task dev:redeploy
```
This rebuilds the image, loads it into the kind cluster, and rolls the deployment.

**kubeconfig not found**
The controller looks for `kubeconfigPath` from `config/overlays/dev/config.yaml`. If the path doesn't exist the pod will crash-loop. For local development, either remove the field (falls back to in-cluster config) or mount a valid kubeconfig at the configured path.

## Development

```bash
task build       # Build the binary
task test        # Run tests
task lint        # Run linter
task generate    # Run code generation (deepcopy, defaults)
task manifests   # Generate CRD, RBAC, and webhook manifests
task dev:setup   # Bootstrap kind cluster and deploy
task dev:redeploy  # Rebuild image and roll pods
```
