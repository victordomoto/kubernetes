# CKA Practice Questions

This file collects hands-on practice tasks organized by the CKA competency domains. Attempt each task before opening its answer. Keep new questions under the closest matching domain so this document can grow without becoming topic-specific.

Unless a task says otherwise, assume `kubectl` and any named tools are installed and the current kubeconfig points to the practice cluster. Tasks that mention preloaded releases, files, or cluster conditions require an instructor-provided lab setup. Verify results in the requested namespace and cluster context.

## Cluster Architecture, Installation & Configuration

<details>
<summary>Helm</summary>

### 1. Install with the right release and values

For chart-repository questions, use the Bitnami repository and `bitnami/nginx` unless the task says otherwise. Inspect the chart's available values and versions instead of relying on remembered defaults. Do not edit Helm-managed Kubernetes objects directly when the task asks you to change chart configuration.

Add and refresh the chart repository. Find the NGINX chart and inspect its configurable values. Install a release named `edge` into a new namespace named `frontend` with:

- 2 replicas
- a `ClusterIP` Service
- a values file named `frontend-values.yaml`

Verify the Helm release, its effective values, the rendered manifest, and the Kubernetes objects in `frontend`. The namespace must not need to exist before your install command runs.

**Pass criteria:** release `edge` is deployed in `frontend`; it has 2 replicas and a `ClusterIP` Service; the values file is used; verification is scoped to the correct namespace.

<details>
<summary>Answer</summary>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/nginx
helm show values bitnami/nginx

cat > frontend-values.yaml <<'EOF'
replicaCount: 2
service:
	type: ClusterIP
EOF

helm install edge bitnami/nginx \
	--namespace frontend \
	--create-namespace \
	--values frontend-values.yaml

helm status edge -n frontend
helm get values edge -n frontend --all
helm get manifest edge -n frontend
kubectl get deployments,services -n frontend
```

</details>

### 2. Same release name, different namespace

Starting from Question 1, install another NGINX release also named `edge`, this time in a new namespace named `staging`, with 1 replica. Do not change or uninstall the `frontend/edge` release.

Show both releases in one Helm listing and verify each namespace has its own release and workload.

**Pass criteria:** `edge` exists in both `frontend` and `staging`; the first release still has 2 replicas and the second has 1.

<details>
<summary>Answer</summary>

```bash
helm install edge bitnami/nginx \
	--namespace staging \
	--create-namespace \
	--set replicaCount=1

helm list --all-namespaces --filter '^edge$'
kubectl get deployments -n frontend
kubectl get deployments -n staging
```

Helm release names are unique within a namespace, so the same name can be used in both namespaces.

</details>

### 3. Values precedence under pressure

Create these two files:

`base-values.yaml`:

```yaml
replicaCount: 2
service:
	type: ClusterIP
```

`prod-values.yaml`:

```yaml
replicaCount: 3
```

Render the chart for a release named `preview` using both files, in that order, and a command-line override setting `replicaCount` to `4`. Do not install a release.

**Pass criteria:** the rendered workload has 4 replicas, its Service remains `ClusterIP`, and no `preview` release is created.

<details>
<summary>Answer</summary>

```bash
cat > base-values.yaml <<'EOF'
replicaCount: 2
service:
	type: ClusterIP
EOF

cat > prod-values.yaml <<'EOF'
replicaCount: 3
EOF

helm template preview bitnami/nginx \
	--values base-values.yaml \
	--values prod-values.yaml \
	--set replicaCount=4

helm list --all-namespaces --filter '^preview$'
```

Later values files override earlier files, and `--set` takes precedence over both. `helm template` renders locally and does not install a release.

</details>

### 4. Upgrade without losing existing configuration

Starting from Question 1, upgrade `frontend/edge` to 4 replicas. Preserve its existing chart values, including the `ClusterIP` Service setting. Do not create a second release.

Verify the release and workload after the upgrade.

**Pass criteria:** `frontend/edge` has 4 replicas, the Service remains `ClusterIP`, and the release name and namespace are unchanged.

<details>
<summary>Answer</summary>

```bash
helm upgrade edge bitnami/nginx \
	--namespace frontend \
	--reuse-values \
	--set replicaCount=4

helm status edge -n frontend
helm get values edge -n frontend --all
kubectl get deployment,service -n frontend
```

</details>

### 5. Roll back the correct revision

Starting from Question 4, inspect the release history and return `frontend/edge` to the configuration it had immediately after Question 1. Do not uninstall or reinstall it.

Verify the workload configuration and inspect the release history again.

**Pass criteria:** the workload is back to 2 replicas; Helm history shows the rollback as a new revision rather than erasing earlier revisions.

<details>
<summary>Answer</summary>

```bash
helm history edge -n frontend
helm rollback edge 1 -n frontend

helm status edge -n frontend
helm history edge -n frontend
kubectl get deployment -n frontend
```

Question 1 created revision 1. A rollback creates a new revision using the selected revision's configuration; it does not erase the intervening history.

</details>

### 6. Make an install command safe to rerun

Write and run a single Helm command for a release named `api` using `bitnami/nginx`, targeting namespace `api-team`, with 2 replicas. It must work whether the release and namespace are absent or the release already exists. Run it twice.

**Pass criteria:** both runs succeed; there is exactly one `api` release in `api-team`, and its workload has 2 replicas.

<details>
<summary>Answer</summary>

```bash
helm upgrade --install api bitnami/nginx \
	--namespace api-team \
	--create-namespace \
	--set replicaCount=2

helm upgrade --install api bitnami/nginx \
	--namespace api-team \
	--create-namespace \
	--set replicaCount=2

helm list -n api-team
kubectl get deployments -n api-team
```

</details>

### 7. Install from a local chart archive

The lab provides a packaged chart at `/tmp/charts/web-1.2.0.tgz` and a values file at `/tmp/web-values.yaml`. Install it as release `web` in namespace `web-team`, creating the namespace if needed. Then verify the release and the resources it created.

**Pass criteria:** the local archive is used (not a repository chart with the same name); the release is in `web-team`; the provided values are applied.

<details>
<summary>Answer</summary>

```bash
helm install web /tmp/charts/web-1.2.0.tgz \
	--namespace web-team \
	--create-namespace \
	--values /tmp/web-values.yaml

helm status web -n web-team
helm get values web -n web-team --all
kubectl get all -n web-team
```

</details>

### 8. Diagnose a release whose Pod is not Ready

The lab provides a release named `broken-web` in namespace `incident`. Helm reports the release as deployed, but its Pod is not Ready. Without uninstalling the release, determine why the Pod is unhealthy and report the evidence you used.

**Pass criteria:** identify the actual workload or container failure from Kubernetes status, events, or logs; distinguish Helm release status from Pod readiness; leave the release installed.

<details>
<summary>Answer</summary>

```bash
helm status broken-web -n incident
helm get all broken-web -n incident
kubectl get pods -n incident -o wide
kubectl describe pod <pod-name> -n incident
kubectl logs <pod-name> -n incident --all-containers
kubectl get events -n incident --sort-by=.metadata.creationTimestamp
```

Use the Pod's conditions, container state, events, and logs to determine the specific cause; it depends on the supplied lab state. Helm's `deployed` status describes the release operation, not whether every Pod is Ready. Do not uninstall the release.

</details>

### 9. Select a chart version deliberately

Search the chart repository for all available versions of NGINX. Install release `pinned-web` in namespace `pinned` using a specific chart version shown by the search results. Record the selected chart version and verify the release is installed from that version.

**Pass criteria:** a version is explicitly selected during install; the installed release is in `pinned`; the chosen chart version is recorded rather than inferred from the application version.

<details>
<summary>Answer</summary>

```bash
helm search repo bitnami/nginx --versions
```

Choose a chart version from the results, then substitute it for `<CHART_VERSION>`:

```bash
helm install pinned-web bitnami/nginx \
	--namespace pinned \
	--create-namespace \
	--version <CHART_VERSION>

helm list -n pinned
helm status pinned-web -n pinned
```

Record the selected chart version from the `CHART` column in `helm list` (for example, `nginx-<CHART_VERSION>`). The chart version is distinct from the application version.

</details>

</details>

<details>
<summary>Kustomize</summary>

Use the paths and cluster context stated in each task. Render a kustomization before applying when you need to confirm what resources or changes it contains.

### 1. Render and apply a supplied kustomization

The lab provides a complete kustomization at `/opt/kustomize/base`. Inspect its resources, render the final manifests, apply them, and verify that the resources were created in the namespace specified by the manifests.

**Pass criteria:** the rendered output contains the expected resources; the build is applied with Kustomize; the resources are verified in their intended namespace.

<details>
<summary>Answer</summary>

```bash
kubectl config current-context
kubectl kustomize /opt/kustomize/base
kubectl apply -k /opt/kustomize/base
```

Read the rendered metadata to identify the namespace and resource names, then verify the relevant objects. For example, if the resources are namespaced in `app`:

```bash
kubectl get all -n app
kubectl get events -n app --sort-by=.metadata.creationTimestamp
```

Rendering alone does not apply anything. Use `-k` for the Kustomize directory.

</details>

### 2. Create an environment overlay

The lab provides `app/base`, containing a Deployment named `web` with a container named `nginx` using image `nginx:1.25`, and a Service. Create `app/overlays/staging` so it reuses the base, sets the namespace to `staging`, sets the Deployment to 3 replicas, and changes the image tag to `1.27`. Render and apply the staging overlay without changing the base.

**Pass criteria:** only the staging overlay is applied; the Deployment has 3 replicas and uses `nginx:1.27`; its resources are in namespace `staging`.

<details>
<summary>Answer</summary>

Create `app/overlays/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
	- ../../base
namespace: staging
replicas:
	- name: web
		count: 3
images:
	- name: nginx
		newTag: "1.27"
```

Then render, apply, and verify:

```bash
kubectl kustomize app/overlays/staging
kubectl apply -k app/overlays/staging
kubectl get deployment,service -n staging
kubectl get deployment web -n staging -o yaml
```

</details>

### 3. Patch a Deployment through an overlay

The lab provides `app/base` and an overlay at `app/overlays/production` whose `kustomization.yaml` already references the base. Add a patch that gives the `nginx` container in Deployment `web` a CPU request of `100m` and a memory request of `128Mi`. Do not edit the base Deployment. Render and apply the production overlay.

**Pass criteria:** the patch is applied only through the production overlay; the rendered Deployment contains both requests; the base manifest remains unchanged.

<details>
<summary>Answer</summary>

Create `app/overlays/production/resources-patch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: web
spec:
	template:
		spec:
			containers:
				- name: nginx
					resources:
						requests:
							cpu: 100m
							memory: 128Mi
```

Add the patch to the existing `patches` list in `app/overlays/production/kustomization.yaml`, preserving its current entries:

```yaml
patches:
	- path: resources-patch.yaml
```

Render, apply, and verify:

```bash
kubectl kustomize app/overlays/production
kubectl apply -k app/overlays/production
kubectl get deployment web -n production -o yaml
```

</details>

### 4. Generate a ConfigMap and update its workload reference

The lab provides `/opt/kustomize/config/deployment.yaml`, a Deployment in namespace `app` that references a ConfigMap named `web-config` using `envFrom.configMapRef`. Update the kustomization in that directory to include the Deployment and generate `web-config` with the literal `APP_MODE=production`. Apply the build and verify both the ConfigMap and the Deployment reference.

**Pass criteria:** the ConfigMap contains `APP_MODE=production`; the Deployment references the generated ConfigMap name; both are in namespace `app`.

<details>
<summary>Answer</summary>

Set `/opt/kustomize/config/kustomization.yaml` to include the resource and generator:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
	- deployment.yaml
namespace: app
configMapGenerator:
	- name: web-config
		literals:
			- APP_MODE=production
```

Render and apply the directory:

```bash
kubectl kustomize /opt/kustomize/config
kubectl apply -k /opt/kustomize/config
kubectl get configmaps -n app
kubectl get deployment -n app -o yaml
```

The generated ConfigMap name normally has a content hash suffix. Kustomize updates recognized references in the Deployment during the build.

</details>

### 5. Include a manifest missing from the build

The lab provides `/opt/kustomize/fix/kustomization.yaml` and a `network-policy.yaml` manifest in the same directory. The NetworkPolicy is not present in the rendered output. Find and correct the kustomization so the policy is included, then render, apply, and verify it in namespace `secure`.

**Pass criteria:** the rendered output includes the NetworkPolicy; it is applied; `deny-all` exists in namespace `secure`.

<details>
<summary>Answer</summary>

Add `network-policy.yaml` to the existing `resources` list in `/opt/kustomize/fix/kustomization.yaml`, preserving any resources already listed:

```yaml
resources:
	- network-policy.yaml
```

Then confirm the build includes it and apply:

```bash
kubectl kustomize /opt/kustomize/fix
kubectl apply -k /opt/kustomize/fix
kubectl get networkpolicy deny-all -n secure
```

</details>

### 6. Delete resources from a dedicated kustomization

The lab's dedicated cleanup environment is defined entirely by `/opt/kustomize/cleanup`, and all of its namespaced resources belong to namespace `lab`. Remove the resources managed by this kustomization and verify they are gone. Do not run this against a shared base or an environment containing resources that should remain.

**Pass criteria:** resources from the dedicated kustomization are deleted from `lab`; no unrelated namespace or resource is targeted.

<details>
<summary>Answer</summary>

```bash
kubectl kustomize /opt/kustomize/cleanup
kubectl delete -k /opt/kustomize/cleanup
kubectl get all,configmaps,secrets,networkpolicies -n lab
```

The render step lets you inspect the deletion target. `kubectl delete -k` deletes the resources listed or generated by that build; it does not remove the manifests from disk.

</details>
</details>
