# INITIALIZATION REQUIRED

If `.claude/.initialized` does not exist in this repository, **invoke the `/init-service` skill immediately** as your first action — before responding to any user message. The SessionStart hook injects a reminder when this is the case. Do not ask for permission; just run it.

---

# controller-template

A template repository for building Kubernetes controllers that connect to the Milo control plane. Based on kubebuilder v4, modeled after the `billing` service conventions.

## Architecture

- **CRD-based**: Uses kubebuilder v4 CRDs (not aggregated API server)
- **Controller-runtime**: Reconciler for lifecycle management
- **Milo control plane aware**: `KubeconfigPath` config field allows pointing the controller at the Milo API server
- **Webhooks**: Validating and defaulting webhook support wired up

## API Group

- Group: `example.miloapis.com`
- Version: `v1alpha1`
- Resources: `Resource` (rename when forking this template)

## Repo Layout

```
controller-template/
├── cmd/controller-template/main.go  # Binary entrypoint
├── api/v1alpha1/                     # CRD type definitions
├── internal/
│   ├── config/                       # Operator configuration
│   └── controller/                   # Reconcilers
├── config/                           # Kustomize manifests
│   ├── base/                         # Core resources
│   ├── components/                   # Optional components
│   └── overlays/                     # Environment-specific
├── ui/                               # Remix UI (React + datum-ui)
│   ├── app/
│   │   ├── components/AppLayout.tsx  # Sidebar + breadcrumb layout
│   │   ├── lib/                      # k8s.server, kubeconfig.server, types, format
│   │   ├── routes/                   # File-based Remix routes
│   │   └── styles/index.css          # Tailwind + datum-ui theme
│   ├── Dockerfile
│   └── package.json
├── hack/                             # Scripts and boilerplate
└── test/e2e/                         # Chainsaw E2E tests
```

## Using This Template

When forking this template for a new service:

1. Replace `controller-template` → `your-service-name` throughout
2. Replace `example.miloapis.com` → `your-group.miloapis.com`
3. Replace `Resource` / `resource` → your CRD kind
4. Replace `ControllerTemplateOperator` → `YourServiceOperator` in `internal/config/config.go`
5. Update `go.mod` module path: `go.miloapis.com/your-service-name`
6. Run `task generate && task manifests` to regenerate code and manifests
7. Update `config/base/manager/config.yaml` and `config/overlays/dev/config.yaml`

## Connecting to the Milo Control Plane

The operator config supports a `kubeconfigPath` field that points at Milo's API server:

```yaml
apiVersion: apiserver.config.miloapis.com/v1alpha1
kind: ControllerTemplateOperator
metricsServer:
  bindAddress: "0"
kubeconfigPath: /etc/milo/kubeconfig
```

When `kubeconfigPath` is empty, the controller falls back to in-cluster config, which is the default for development against a local kind cluster.

## Reconcile errors

Return an error from `Reconcile` only when a retry can fix it. controller-runtime retries every error with backoff and counts it, so an error that no retry can fix repeats for ever as a steady error rate.

- **A failure no retry can fix**, such as input that can never succeed or an object a person must act on: set a condition that names the case and return `reconcile.TerminalError(err)`. It is logged and counted once, and not retried.
- **A wait for something else**, such as another object becoming ready or dependents going away: set a condition, watch what you wait for, and return no error. When you cannot watch it, return `ctrl.Result{RequeueAfter: <poll interval>}`. Do not use `Result.Requeue`, which controller-runtime deprecates. [resend-provider's `finalizer_wait.go`](https://github.com/milo-os/resend-provider/blob/6c8e2b0745735dfffe1dd469278ae19ab03abc32/internal/controller/finalizer_wait.go) does this for a finalizer.
- **Anything else**, such as a timeout, a conflict or a server error: return the error, and controller-runtime retries it.

## Verification Commands

```bash
# Controller
task build                    # Build binary
task test                     # Run tests
task lint                     # Run linter
task generate                 # Run code generation
task manifests                # Generate CRD/RBAC/webhook manifests
task dev:setup                # Bootstrap kind cluster + deploy
task dev:redeploy             # Rebuild and redeploy

# UI
task ui:install               # Install UI dependencies (pnpm)
task ui:dev                   # Start dev server at http://localhost:3000
task ui:build                 # Production build
task ui:type-check            # TypeScript check
```
