# Custom Resource Definitions and Operators

Kubernetes has a built-in API for resources such as Pods, Deployments, and
Services. A **CustomResourceDefinition (CRD)** extends that API with a new
resource type. An **operator** is a controller that watches custom resources
and takes action to make the cluster match the requested state.

For the CKA, be ready to inspect, install, configure, and troubleshoot an
operator and its custom resources. Use the manifests, chart, namespace, and
versions specified in the task; operator-specific fields and install steps
vary.

## Core terms

| Term | Meaning |
|---|---|
| Custom resource (CR) | An instance of a custom API type, stored and served by the Kubernetes API server. |
| CustomResourceDefinition (CRD) | A cluster-scoped API object that registers a custom resource type with the API server. |
| Controller | A control loop that watches resources and reconciles actual state toward desired state. |
| Operator | A controller, usually paired with one or more CRDs, that automates the operational tasks for an application or service. |
| API group and version | Identify the custom API, for example `database.example.com/v1`. |
| Kind and resource name | The Kind is the type, for example `Database`; the resource name identifies one instance, for example `orders-db`. |

A CRD makes a type available through the Kubernetes API, but does not by itself
run an application or implement the type's behavior. The operator/controller
provides that behavior. A CR can be accepted by the API server even if its
operator is absent or not functioning; it may then remain unreconciled.

## How they fit together

1. Install the CRD. The API server registers the custom type and validates
   requests according to its schema.
2. Install and configure the operator/controller. It typically runs as a
   Deployment with a ServiceAccount and RBAC permissions.
3. Create a custom resource of the registered type.
4. The controller watches the resource and reconciles related objects, such as
   Deployments, Services, or persistent storage.
5. Inspect the custom resource's status, generated resources, events, and
   controller logs to verify the result.

The controller's reconciliation loop is usually asynchronous and repeats. A
successful `kubectl apply` means the API server accepted the object; it does
not prove that reconciliation completed or that the application is healthy.

## What a CRD defines

A CRD specifies the custom API's group, names, scope, versions, and schema.
Here is a minimal namespaced `Widget` type:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
    shortNames:
    - wdg
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
            required:
            - replicas
        required:
        - spec
```

The matching custom resource uses the API group/version and Kind from the
definition:

```yaml
apiVersion: example.com/v1
kind: Widget
metadata:
  name: sample
  namespace: default
spec:
  replicas: 2
```

Key details:

- The CRD name is `<plural>.<group>`: `widgets.example.com`.
- The custom resource `apiVersion` is `<group>/<version>`: `example.com/v1`.
- `scope: Namespaced` means each custom resource belongs to a namespace.
  `scope: Cluster` means it does not.
- The CRD itself is cluster-scoped regardless of the custom resource's scope.
- `served: true` enables clients to use that API version. Exactly one version
  has `storage: true`; it is the version used to persist objects. Multiple
  versions may be served.
- The schema validates custom resource fields. With a structural schema,
  unknown fields are normally pruned unless the schema explicitly preserves
  them. Do not assume arbitrary fields will be stored.
- A `status` field and the CRD's `subresources.status` setting can separate
  controller-reported status from user-supplied spec. Follow the operator's
  documented API.

## Operator behavior

Most operators use the standard Kubernetes controller pattern:

- **Watch** custom resources and relevant dependent resources.
- **Reconcile** the current state with the desired state in each resource's
  `spec`.
- **Report** progress or observed state in `status`, conditions, events, or
  logs (depending on the operator).
- **Retry** reconciliation when watched objects change or an operation fails.

The exact meaning of `spec` fields and `status` conditions belongs to each
operator. Read its documentation or inspect the CRD schema rather than
guessing. Deleting a custom resource may trigger cleanup of resources managed
by the operator, but the cleanup behavior is operator-specific.

Some operators also use admission webhooks for validation or defaulting,
conversion webhooks to translate between API versions, and finalizers to
complete cleanup before an object is removed. These components can affect
admission and deletion; diagnose them through the operator's installation
resources, events, and logs.

## Install and configure

Follow the task's installation method. Operators are commonly distributed as
plain YAML, Helm charts, or vendor-specific bundles. A typical manifest-based
installation looks like:

```bash
# Inspect the supplied files, then register the API before using its type
kubectl apply -f operator/crds/
kubectl wait --for=condition=Established \
  crd/widgets.example.com --timeout=60s

# Install the controller and supporting resources
kubectl apply -f operator/

# Check the operator and its available API
kubectl get deployments,pods -n operators
kubectl get crd widgets.example.com
kubectl api-resources --api-group=example.com

# Validate and create an instance after the type is registered
kubectl apply --dry-run=server -f widget.yaml
kubectl apply -f widget.yaml
```

These are illustrative paths and names, not a universal operator layout.
Some packages include CRDs in the same directory or require configuration
before installation. Use the package's instructions and task requirements.
When using Helm, inspect the chart's values and verify its release and
namespace; some charts install CRDs separately from the release's managed
resources.

Check the operator's ServiceAccount permissions if it cannot watch or manage
resources. Its RBAC must allow the required verbs (`get`, `list`, `watch`,
`create`, `update`, `patch`, or `delete`) on the correct API groups and
resources. Do not grant broader permissions than the supplied configuration
or task requires.

## Inspect custom APIs and resources

```bash
# List CRDs and see which custom resource types exist
kubectl get crd
kubectl get crd widgets.example.com -o yaml
kubectl describe crd widgets.example.com
kubectl api-resources --api-group=example.com
kubectl api-versions

# Inspect a custom resource (use its plural resource name)
kubectl get widgets -n default
kubectl get widgets sample -n default -o yaml
kubectl describe widget sample -n default

# Discover the fields documented by the API schema
kubectl explain widget
kubectl explain widget.spec
kubectl explain widget.spec.replicas
kubectl explain widget --api-version=example.com/v1
```

`kubectl api-resources` reports the resource name, Kind, API group, and whether
the resource is namespaced. Use the plural resource name from that output for
`kubectl get`; `kubectl explain` can use the Kind.

## Troubleshooting checklist

### `no matches for kind ...` or `the server doesn't have a resource type`

1. Confirm the current context and cluster.
2. Check that the CRD exists: `kubectl get crd`.
3. Check that its name, group, and served version match the custom resource's
   `apiVersion` and `kind`.
4. Wait for the CRD's `Established` condition before applying custom resources.
5. Check `kubectl api-resources --api-group=<group>` for the registered type.

### The custom resource is rejected

- Check the API version, Kind, required fields, types, and allowed values
  against the CRD schema (`kubectl explain` and `kubectl get crd -o yaml`).
- Check the namespace for namespaced resources.
- Inspect the full API server error; schema validation errors commonly name
  the invalid or missing field.

### The custom resource exists but nothing happens

- Check that the operator Deployment and Pods are running in the expected
  namespace.
- Inspect its logs and events:

  ```bash
  kubectl logs deployment/<operator-deployment> -n <operator-namespace>
  kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
  ```

- Inspect the custom resource's `status` and conditions, if defined.
- Check the operator's ServiceAccount, Role/ClusterRole, and
  RoleBinding/ClusterRoleBinding for required permissions.
- Confirm the controller watches the resource's namespace and API group.
- Check dependent resources it should have created, such as Pods or
  Deployments, and troubleshoot those using the usual Kubernetes workflow.

### Deletion appears stuck

Inspect the object's finalizers and the controller/webhook health. A finalizer
can keep an object in `Terminating` while a controller performs cleanup.
Do not remove finalizers blindly: doing so can leave dependent resources behind
or skip required cleanup.

## Important safety and version notes

- Deleting a CRD removes the custom API type and deletes the custom resources
  stored under that type. Do not delete a CRD as a troubleshooting shortcut.
- Upgrading an operator and upgrading its CRDs are related but distinct
  operations. Follow the release's documented upgrade order and compatibility
  requirements.
- A CRD can serve multiple API versions; schema conversion and storage-version
  migration are not automatic consequences of changing an operator image.
- Webhooks can prevent API operations when their service, certificates, or
  endpoints are unavailable. Inspect webhook configurations and operator
  Pods/events before changing or removing them.
- Operator behavior, custom fields, and cleanup guarantees are not standardized
  across vendors. Consult the supplied CRD, chart values, and official
  instructions for the exact operator in the task.

## CKA quick checklist

1. Read the required operator, version, namespace, and configuration carefully.
2. Confirm cluster context and inspect existing CRDs, resources, and releases
   before installing anything.
3. Install or establish the CRD before applying custom resources.
4. Configure the operator using its documented values or manifests; check its
   Deployment, Pods, and RBAC.
5. Create or update the custom resource with the exact API version and fields.
6. Verify both the custom resource's status and the Kubernetes objects the
   operator manages.
7. If reconciliation fails, inspect events, controller logs, and permissions;
   an accepted custom resource is not proof of success.
8. Avoid deleting CRDs or stripping finalizers unless the task explicitly
   requires it and you understand the consequences.

## Quick reference

| Goal | Command |
|---|---|
| List registered custom APIs | `kubectl get crd` |
| Inspect one CRD | `kubectl describe crd <plural>.<group>` |
| Find resources in an API group | `kubectl api-resources --api-group=<group>` |
| Wait for CRD registration | `kubectl wait --for=condition=Established crd/<name>` |
| List custom resources in all namespaces | `kubectl get <plural> -A` |
| Read a custom resource schema | `kubectl explain <kind> --api-version=<group>/<version>` |
| Inspect an instance and its status | `kubectl get <plural> <name> -n <namespace> -o yaml` |
| Read operator logs | `kubectl logs deployment/<name> -n <operator-namespace>` |

Official references:

- [Custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Extend the Kubernetes API with the CustomResourceDefinition API](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
