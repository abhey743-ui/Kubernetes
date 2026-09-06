# Helm — The Kubernetes Package Manager

## 1. What is Helm?

Helm is a package manager for Kubernetes. It lets you define, install, and upgrade even the most complex Kubernetes applications using a single unit called a **chart**.

Think of it like `apt`/`yum` for Linux, or `npm` for Node.js — but instead of installing binaries, Helm installs collections of Kubernetes manifests (Deployments, Services, ConfigMaps, Ingress, etc.) as one versioned, configurable package.

**Why Helm exists:**
- Raw Kubernetes YAML gets repetitive and hard to manage across environments (dev/staging/prod).
- Helm templates YAML so you can reuse the same chart with different config values.
- It tracks "releases" so you can upgrade, rollback, or uninstall an entire application as one unit instead of managing dozens of individual manifests by hand.

---

## 2. Core Concepts

| Term | Meaning |
|---|---|
| **Chart** | A package of pre-configured Kubernetes resources (templates + metadata). The unit of distribution in Helm. |
| **Release** | A specific instance of a chart deployed into a cluster. You can install the same chart multiple times, each becoming a separate release. |
| **Repository** | A place where charts are collected and shared (like a package registry, e.g. Artifact Hub, Bitnami, or your own OCI registry). |
| **Values** | Configuration passed into a chart's templates (`values.yaml` or `--set` flags), letting one chart behave differently per environment. |
| **Templates** | Go-template files inside a chart that generate final Kubernetes manifests once values are injected. |

---

## 3. Architecture

### Helm 3 architecture (current — no Tiller)

Older Helm 2 used a server-side component called **Tiller** running inside the cluster, which was a security concern (it had broad cluster permissions). **Helm 3 removed Tiller entirely.**

```
┌────────────────────┐
│   Helm CLI (client) │   <- runs on your laptop/CI
└─────────┬───────────┘
          │ talks directly to
          ▼
┌────────────────────┐
│ Kubernetes API      │   <- uses your kubeconfig / RBAC
│ Server              │
└─────────┬───────────┘
          │ creates/updates
          ▼
┌────────────────────┐
│ Cluster resources   │   (Deployments, Services, etc.)
└────────────────────┘
          │
          ▼
┌────────────────────┐
│ Release metadata    │   <- stored as a Secret (default)
│ (stored in-cluster) │      in the release's namespace
└────────────────────┘
```

Key architectural points:

1. **Client-only design** — Helm is just a CLI binary. It authenticates to Kubernetes the same way `kubectl` does (via `~/.kube/config`), and issues API requests directly. There's no in-cluster server component to secure or maintain.

2. **Release storage** — Helm stores the state of each release (chart version, values used, manifest rendered) as a Kubernetes **Secret** object (configurable to ConfigMap or SQL) in the namespace of the release. This is how `helm history`, `helm rollback`, and `helm status` work — Helm reads this stored state rather than keeping its own database.

3. **Templating engine** — Charts use Go's `text/template` syntax plus the **Sprig** function library. When you run `helm install`, Helm:
   - Loads the chart's `values.yaml` (plus any overrides you pass).
   - Renders every file in `templates/` through the Go template engine, substituting `{{ .Values.xxx }}` references.
   - Concatenates the rendered YAML documents.
   - Sends them to the Kubernetes API as a single atomic operation.

4. **Chart repositories** — A repository is just an HTTP(S) server hosting an `index.yaml` file plus packaged charts (`.tgz`). Helm can also pull charts from **OCI registries** (Docker/OCI-compatible registries like ghcr.io, ECR, ACR) since Helm 3.8+.

5. **Hooks** — Charts can define lifecycle hooks (`pre-install`, `post-install`, `pre-upgrade`, `pre-delete`, etc.) as annotated Kubernetes Jobs/Pods that Helm runs at specific points in the release lifecycle (e.g. running a DB migration before upgrading an app).

---

## 4. Anatomy of a Chart

```
mychart/
├── Chart.yaml           # Metadata: name, version, description, dependencies
├── values.yaml           # Default configuration values
├── charts/               # Sub-charts (dependencies) bundled in-line
├── templates/            # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl      # Reusable template snippets/functions
│   └── NOTES.txt         # Printed to user after install/upgrade
├── .helmignore           # Files to exclude when packaging
└── templates/tests/      # Optional helm test definitions
```

**Chart.yaml** example:
```yaml
apiVersion: v2
name: mychart
description: A sample Helm chart
version: 0.1.0        # Chart version
appVersion: "1.16.0"  # Version of the app it deploys
```

**values.yaml** example:
```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
service:
  type: ClusterIP
  port: 80
```

**templates/deployment.yaml** example (uses the values above):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-mychart
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: mychart
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Built-in objects you can reference in templates: `.Release` (name, namespace, revision), `.Chart` (chart metadata), `.Values` (merged values), `.Files`, `.Capabilities`.

---

## 5. Installation / Setup

### Install the Helm CLI

**macOS (Homebrew):**
```bash
brew install helm
```

**Linux (script installer):**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Linux (apt, Debian/Ubuntu):**
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

**Windows (Chocolatey / Scoop):**
```powershell
choco install kubernetes-helm
# or
scoop install helm
```

**Verify:**
```bash
helm version
```

Helm needs a working `kubectl` context pointing at your cluster — it reuses your existing `~/.kube/config`, so if `kubectl get nodes` works, Helm will work too.

---

## 6. Everyday Commands

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress

# Install a chart as a new release
helm install my-release bitnami/nginx

# Install with custom values
helm install my-release bitnami/nginx -f custom-values.yaml
helm install my-release bitnami/nginx --set replicaCount=3

# List releases
helm list
helm list --all-namespaces

# Check status / rendered manifest
helm status my-release
helm get manifest my-release

# Upgrade a release (change values or chart version)
helm upgrade my-release bitnami/nginx --set replicaCount=5

# Upgrade or install if not present (idempotent, great for CI/CD)
helm upgrade --install my-release bitnami/nginx

# Roll back to a previous revision
helm history my-release
helm rollback my-release 1

# Uninstall
helm uninstall my-release

# Create a new chart skeleton
helm create mychart

# Render templates locally without installing (debugging)
helm template mychart
helm install my-release ./mychart --dry-run --debug

# Package a chart for distribution
helm package mychart

# Lint a chart for errors
helm lint mychart
```

---

## 7. Values Override Precedence (lowest → highest)

1. Chart's own `values.yaml`
2. Parent chart's `values.yaml` (if used as a sub-chart/dependency)
3. A values file passed with `-f custom-values.yaml` (later files override earlier ones)
4. `--set key=value` flags on the command line (highest precedence)

---

## 8. Dependencies (Sub-charts)

A chart can depend on other charts (e.g. an app chart depending on a Redis chart), declared in `Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

Then pull them in:
```bash
helm dependency update
```
This downloads the dependency charts into the `charts/` folder so they get installed together as one release.

---

## 9. Helm Testing

Charts can include test hooks (Pods annotated `"helm.sh/hook": test`) that verify a release works after install:
```bash
helm test my-release
```

---

## 10. Best Practices

- **Pin chart versions** in production (`helm install app repo/chart --version 1.2.3`) rather than always pulling `latest`.
- **Separate values files per environment** (`values-dev.yaml`, `values-prod.yaml`) instead of branching logic inside templates.
- **Use `helm diff` plugin** (`helm plugin install https://github.com/databus23/helm-diff`) to preview changes before `helm upgrade`.
- **Store charts in version control**, and use `helm lint` + `helm template` in CI to catch template errors before deploying.
- **Avoid storing secrets in `values.yaml`** in plaintext — use tools like Helm Secrets, Sealed Secrets, or External Secrets Operator instead.
- **Use `--atomic`** on `helm upgrade`/`install` so a failed deploy automatically rolls back:
  ```bash
  helm upgrade --install my-release ./mychart --atomic
  ```

---

## 11. Helm vs. Plain `kubectl apply`

| | `kubectl apply -f` | Helm |
|---|---|---|
| Templating/reuse across environments | No (manual copy-paste or Kustomize needed) | Yes, built-in |
| Versioned releases & rollback | No | Yes (`helm rollback`) |
| Packaging/distribution | No | Yes (chart repos, OCI registries) |
| Dependency management | No | Yes (sub-charts) |
| Atomic multi-resource install | Partial | Yes |

Helm and `kubectl`/Kustomize aren't mutually exclusive — many teams use Helm for third-party software (databases, ingress controllers, monitoring stacks) and Kustomize or raw manifests for simple internal apps.

---

*Good luck with Helm — once the chart/template/values mental model clicks, it makes Kubernetes app management dramatically less painful.*
