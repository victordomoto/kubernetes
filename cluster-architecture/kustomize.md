# Kustomize for the CKA

Kustomize customizes Kubernetes manifests without a template language. It composes resources and applies changes such as namespaces, image updates, replicas, patches, and generated ConfigMaps or Secrets. Kustomize is built into `kubectl`, so the common CKA workflow uses `kubectl kustomize` to render and `kubectl apply -k` to apply.

For the CKA, focus on locating a kustomization directory, understanding what it renders, applying it to the intended cluster, and making a small requested change through the correct overlay or patch.

---

## Core terms

| Term | Meaning |
|---|---|
| Resource | A Kubernetes manifest included in a Kustomize build, such as a Deployment or Service. |
| `kustomization.yaml` | The configuration file listing resources and describing customizations. `Kustomization` and `kustomization.yml` are also recognized names. |
| Base | A reusable set of resources and configuration, commonly referenced by one or more overlays. |
| Overlay | A kustomization that references a base and adds environment- or task-specific changes. |
| Transformer | A built-in customization such as setting a namespace, changing an image, or adjusting replicas. |
| Patch | A targeted change to one or more fields in a resource. |

Kustomize renders ordinary Kubernetes objects. It does not create Helm-style releases, revisions, or rollback history.

---

## Common workflow

### 1. Inspect the directory

```bash
ls -R /path/to/kustomize
cat /path/to/kustomize/kustomization.yaml
```

Find the kustomization file and the resources or child directories it references. Paths in a kustomization are relative to that file's directory. Read the referenced manifests and patches before applying if the task requires specific changes.

### 2. Render before applying

```bash
kubectl kustomize /path/to/kustomize
```

This builds and prints the final YAML locally; it does not change the cluster. Review the output for the expected resource names, namespace, image, replica count, and other task requirements.

### 3. Apply the build

```bash
kubectl apply -k /path/to/kustomize
```

`-k` tells `kubectl` to build the directory with Kustomize. It is different from `-f`, which applies a manifest file or directory of manifest files. Use the path provided by the task exactly.

### 4. Verify the cluster state

```bash
kubectl get all -n <namespace>
kubectl get deployments,services,configmaps,secrets -n <namespace>
kubectl describe deployment <name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Use the resource types and namespace relevant to the task; `kubectl get all` does not include every Kubernetes resource type. Check actual cluster objects after applying, not only the rendered output.

---

## Kustomization file basics

A minimal `kustomization.yaml` lists resource manifests. Resource entries can also refer to directories that contain their own kustomization file.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
	- deployment.yaml
	- service.yaml
```

Build and apply from the directory containing this file:

```bash
kubectl kustomize ./app
kubectl apply -k ./app
```

Use `resources` to add existing manifests to the build. A manifest placed in the directory but not listed in `resources` is not included unless it is brought in through another referenced kustomization.

---

## Overlays and environment-specific changes

An overlay references a base and applies only the differences needed for that environment. A common layout is:

```text
app/
	base/
		kustomization.yaml
		deployment.yaml
		service.yaml
	overlays/
		staging/
			kustomization.yaml
			deployment-patch.yaml
```

`app/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
	- deployment.yaml
	- service.yaml
```

`app/overlays/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
	- ../../base
namespace: staging
replicas:
	- name: web
		count: 2
images:
	- name: nginx
		newTag: "1.27"
patches:
	- path: deployment-patch.yaml
```

Build or apply the overlay, not just the base:

```bash
kubectl kustomize app/overlays/staging
kubectl apply -k app/overlays/staging
```

The `namespace`, `replicas`, and `images` entries transform matching resources. The image `name` must match the image name in the original manifest. A transformer only changes resources included in the build.

---

## Patches

Use a patch when a task asks for a field change that is not covered cleanly by a built-in transformer. For example, a strategic-merge patch can change a Deployment's container port or add a field:

`deployment-patch.yaml`:

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
```

The `patches` entry in the overlay shown above applies this patch to the matching Deployment. The patch must identify the resource using its API version, kind, and name. If a patch does not apply, check that the target identity matches the resource after the build inputs are composed.

Kustomize also accepts JSON 6902 patches when an exact path-level change is useful:

```yaml
patches:
	- target:
			version: v1
			kind: Service
			name: web
		patch: |-
			- op: add
				path: /spec/type
				value: NodePort
```

Prefer the simplest patch that satisfies the task, and render the result to confirm it changed only the intended resource.

---

## Generated ConfigMaps and Secrets

Kustomize can generate ConfigMaps and Secrets from literals or files:

```yaml
configMapGenerator:
	- name: app-config
		literals:
			- APP_ENV=staging
		files:
			- app.properties
secretGenerator:
	- name: db-credentials
		literals:
			- username=app
			- password=change-me
```

Generated names usually include a content hash suffix. Kustomize updates references to generated ConfigMaps and Secrets in recognized workload fields when it builds the resources. If the task requires a stable name, a kustomization can disable the suffix:

```yaml
generatorOptions:
	disableNameSuffixHash: true
```

Treat Secret data as sensitive. Base64 encoding in a Secret manifest is not encryption.

---

## Useful commands

```bash
# Render without changing the cluster
kubectl kustomize ./overlay

# Show changes that would be applied
kubectl diff -k ./overlay

# Apply the rendered resources
kubectl apply -k ./overlay

# Delete resources produced by this kustomization
kubectl delete -k ./overlay
```

Use `kubectl diff -k` when available and useful, but do not mistake rendering or diffing for applying. Delete only when the task asks for it; a kustomization may include shared resources.

---

## Troubleshooting

```bash
# Inspect the current context and namespace
kubectl config current-context
kubectl config view --minify

# Render and inspect the exact output
kubectl kustomize /path/to/kustomize

# Check applied objects, status, and events
kubectl get all -n <namespace>
kubectl describe <kind> <name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Common causes of build or apply problems:

- The command points to the wrong directory or the directory has no recognized kustomization file.
- A relative path in `resources` or `patches` is incorrect.
- A resource manifest is not listed in `resources`.
- A patch's kind or name does not match a resource in the build.
- An `images` entry does not match the manifest's original image name.
- The build is correct but was applied to the wrong cluster context or namespace.
- The YAML is valid but the resulting resource is unhealthy; inspect Pod status, events, and logs.

---

## CKA quick checklist

- Confirm the current cluster context and the task's namespace.
- Locate and read the requested kustomization file and its referenced resources.
- Make requested changes in the specified overlay, patch, or manifest; do not edit unrelated files.
- Render with `kubectl kustomize <directory>` and confirm the expected output.
- Apply with `kubectl apply -k <directory>`.
- Verify actual resources and workload readiness with `kubectl` in the correct namespace.
- Remember that Kustomize has no Helm release history or rollback command; make a corrective manifest change and apply it.

### Minimal command sequence

```bash
kubectl config current-context
kubectl kustomize <kustomization-directory>
kubectl apply -k <kustomization-directory>
kubectl get all -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```
