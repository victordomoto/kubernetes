# Helm for the CKA

Helm is a package manager for Kubernetes. It installs and manages Kubernetes resources from reusable packages called **charts**. For the CKA, focus on finding a chart, setting values, installing it into the correct namespace, and inspecting or changing the resulting release.

---

## Core terms

| Term | Meaning |
|---|---|
| Chart | A package containing templates, default values, and chart metadata. |
| Repository | A location from which charts can be searched for and downloaded. |
| Release | A named installation of a chart in a namespace. Installing the same chart twice creates separate releases when the release names differ. |
| Values | Configuration supplied to chart templates, usually through a `values.yaml` file or `--set`. |
| Revision | A numbered version of a release's configuration and rendered resources, used by history and rollback. |

The chart version identifies the chart package. `appVersion` in `Chart.yaml` describes the application version and is informational; it does not necessarily equal the chart version.

---

## Common workflow

### 1. Add and refresh a repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
```

Repositories are local client configuration. Refresh after adding a repository so Helm can discover its latest chart index.

### 2. Find and inspect a chart

```bash
helm search repo nginx
helm search repo nginx --versions
helm show chart bitnami/nginx
helm show values bitnami/nginx
```

Use the chart's documented values keys. Do not guess key names: chart values are specific to each chart.

### 3. Install a release

```bash
helm install web bitnami/nginx --namespace web --create-namespace
```

Install a particular chart version or provide configuration:

```bash
helm install web bitnami/nginx \
	--namespace web \
	--version <chart-version> \
	--values my-values.yaml \
	--set replicaCount=2
```

You can install a chart from a repository, a local chart directory, or a packaged chart archive (`.tgz`). A Helm release is namespaced; use the same namespace when you inspect or update it.

### 4. Verify what was installed

```bash
helm list --namespace web
helm status web --namespace web
helm get values web --namespace web
helm get values web --all --namespace web
helm get manifest web --namespace web
kubectl get all --namespace web
```

`helm get values` shows user-supplied values; add `--all` to include computed values. `helm get manifest` shows the Kubernetes YAML rendered for that release.

---

## Values and overrides

Charts provide defaults in `values.yaml`. Override only the settings you need, preferably in a small file you can inspect and reuse:

```yaml
replicaCount: 2
service:
	type: ClusterIP
```

```bash
helm install web bitnami/nginx -n web -f my-values.yaml
```

Values precedence, from lowest to highest:

1. Chart defaults.
2. Values files supplied with `-f` / `--values` (when multiple files are supplied, later files win).
3. Values supplied with `--set`.

For example, `--set replicaCount=3` overrides `replicaCount` from either the chart defaults or a values file. Quote values when shell parsing or Helm type inference could change their meaning.

---

## Upgrade, rollback, and uninstall

Upgrade an existing release with updated values:

```bash
helm upgrade web bitnami/nginx -n web -f my-values.yaml
```

Use `upgrade --install` when the release may or may not exist:

```bash
helm upgrade --install web bitnami/nginx \
	-n web --create-namespace \
	-f my-values.yaml
```

Review revisions and roll back to a previous one:

```bash
helm history web -n web
helm rollback web <revision> -n web
helm status web -n web
```

Omitting the revision rolls back to the previous revision. A rollback creates a new revision; it does not erase release history.

Remove the release:

```bash
helm uninstall web -n web
```

Uninstall removes the release's managed resources. Do not uninstall unless the task asks for removal.

---

## Useful chart commands

```bash
# Render locally without installing resources
helm template web bitnami/nginx -n web -f my-values.yaml

# Preview an install against the cluster
helm install web bitnami/nginx -n web -f my-values.yaml --dry-run --debug

# Validate a chart directory
helm lint ./my-chart

# Scaffold a chart (know what it does; chart authoring is less common in CKA tasks)
helm create my-chart
```

`helm template` renders manifests locally. A dry run previews the operation; it does not create the release. Use `kubectl` to inspect actual cluster resources after a real install or upgrade.

---

## Release status and troubleshooting

```bash
helm list --all-namespaces
helm status <release> -n <namespace>
helm history <release> -n <namespace>
helm get values <release> -n <namespace> --all
helm get manifest <release> -n <namespace>
kubectl get pods,svc,events -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
```

Check the release name and namespace first. Then inspect its status, values, and rendered manifest, followed by the created Kubernetes objects and namespace events. Helm can report a successful install even when an application Pod is not Ready; use Kubernetes status and events to diagnose workload problems.

---

## CKA quick checklist

- Confirm the active cluster context and the namespace required by the task.
- Search for the requested chart and inspect its values before installing.
- Use the exact release name, chart, namespace, and values specified by the task.
- Use `--create-namespace` when the target namespace may not exist.
- Verify with `helm status` and `kubectl get` in the correct namespace.
- For a change, use `helm upgrade`; for a failed change, inspect `helm history` and use `helm rollback` if appropriate.
- Avoid editing Helm-managed objects directly when the intended change belongs in chart values; a later Helm upgrade may replace manual changes.

### Minimal command sequence

```bash
helm repo add <repo-name> <repo-url>
helm repo update
helm search repo <chart-keyword>
helm show values <repo-name>/<chart>
helm install <release> <repo-name>/<chart> -n <namespace> --create-namespace -f values.yaml
helm status <release> -n <namespace>
kubectl get all -n <namespace>
```
