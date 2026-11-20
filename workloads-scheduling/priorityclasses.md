# PriorityClasses

PriorityClasses let the Kubernetes scheduler distinguish which Pods should
run first and which Pods may be preempted. They are cluster-scoped resources
and are not restricted to one namespace.

For the CKA, know how to create a PriorityClass, assign it to a Pod, set a
default PriorityClass, recognize the built-in system classes, and explain
preemption behavior.

## Core concepts

- Priority affects scheduling and preemption. It does not guarantee that a
  Pod will run.
- A larger integer value means higher priority.
- Priority values must be 32-bit integers.
- Kubernetes reserves values greater than 1 billion for critical system
  components.
- PriorityClass names must be unique in the cluster.
- `PriorityClass` is cluster-scoped, so it has no `metadata.namespace`.
- A Pod selects a PriorityClass with `spec.priorityClassName`.
- The scheduler writes the resolved priority into `status`/the Pod's internal
  priority fields. Do not normally set `spec.priority` manually.

The effective priority of a Pod is:

1. The value of the referenced PriorityClass.
2. Zero when the Pod does not reference a PriorityClass.
3. The value of the default PriorityClass, if one is configured and the Pod
   does not name a class.

## PriorityClass fields

| Field | Purpose |
|---|---|
| `value` | Integer priority. Higher values take precedence. |
| `globalDefault` | Assign this class to Pods that do not specify `priorityClassName`. |
| `description` | Human-readable information about the class. |
| `preemptionPolicy` | Controls whether this Pod can preempt lower-priority Pods. |

`preemptionPolicy` supports:

| Value | Behavior |
|---|---|
| `PreemptLowerPriority` | Default. The Pod may preempt lower-priority Pods to become schedulable. |
| `Never` | The Pod is placed in the scheduling queue without preempting existing Pods. |

A Pod whose priority is too low for available resources is usually not
preempted immediately by the scheduler. A higher-priority Pending Pod may
instead trigger preemption of lower-priority running Pods.

## Create a PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Workloads that should be scheduled before regular workloads"
```

```bash
kubectl apply -f priorityclass.yaml
kubectl get priorityclasses
kubectl get pc
kubectl describe priorityclass high-priority
kubectl delete priorityclass high-priority
```

Create a class with a manifest is safer than relying on generated YAML. The
following command can generate a starting manifest:

```bash
kubectl create priorityclass high-priority \
  --value=100000 \
  --global-default=false \
  --description="High-priority workloads"
```

## Assign a PriorityClass to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx:1.27
    resources:
      requests:
        cpu: "500m"
        memory: "128Mi"
```

The same field belongs in a workload Pod template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      priorityClassName: high-priority
      containers:
      - name: api
        image: nginx:1.27
```

`kubectl set` does not edit `spec.priorityClassName` on an existing Pod as a
normal mutable field. If a task requires changing priority for a workload,
patch or edit the workload template and allow it to create new Pods.

## Default PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 1000
globalDefault: true
description: "Default priority for workloads that do not set one"
```

Important behavior:

- At most one PriorityClass should have `globalDefault: true`.
- Kubernetes rejects adding a second global default after one already exists.
- A global default applies only when the Pod does not name a PriorityClass.
- A Pod that explicitly names a class uses that class even if another class is
  the global default.
- If no global default exists, Pods without a priority class have priority zero.
- Adding or changing a global default does not retroactively change existing
  Pods; it affects subsequent admission.

Check, change, and remove the default setting:

```bash
kubectl get priorityclasses
kubectl describe priorityclass default-priority
kubectl patch priorityclass default-priority \
  -p '{"globalDefault":false}'
```

When patching an existing global default, make sure the replacement default is
valid before removing the old setting.

## Built-in PriorityClasses

Common Kubernetes installations include two system classes:

| Class | Purpose |
|---|---|
| `system-cluster-critical` | Cluster-wide critical components |
| `system-node-critical` | Node-critical components, such as a node agent |

Examples of node-critical components include components required on the node
to keep it functioning. Do not generally assign application workloads to these
classes. Their values are reserved and should not be reused for regular
applications.

List the built-in classes:

```bash
kubectl get priorityclasses
```

Do not assume a cluster has both built-in classes; inspect it. Custom clusters
and distributions can differ.

## Preemption

Preemption is the process where a higher-priority Pending Pod can displace
lower-priority running Pods.

The scheduler:

1. Tries to schedule the Pod normally.
2. If the Pod remains unschedulable, considers lower-priority victim Pods.
3. Finds a node or set of nodes where removing those Pods would make room.
4. Nominates or preempts victims.
5. Re-evaluates scheduling and places the high-priority Pod.

Preemption is not equivalent to guaranteed resource reservation. The scheduler
may need to preempt several Pods, and another higher-priority Pod may change
the result.

The scheduler also considers scheduling constraints such as node affinity,
taints and tolerations, and pod affinity/anti-affinity. It does not preempt
Pods merely because a higher-priority Pod exists; the lower-priority Pods must
be the reason a suitable placement is currently unavailable.

PodDisruptionBudgets and graceful termination influence workload disruption,
but do not turn priority into a capacity guarantee. Check
`PodDisruptionBudget`, `terminationGracePeriodSeconds`, and eviction behavior
when diagnosing workload interruption.

Use `preemptionPolicy: Never` for workloads that should not evict other Pods:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: no-preemption
value: 5000
preemptionPolicy: Never
description: "High-priority queue position without preempting running Pods"
```

## Priority and scheduling order

Priority mainly affects:

- Which Pending Pod is considered first by the scheduler.
- Whether a Pending Pod can preempt running lower-priority Pods.
- Which Pods are selected as victims during preemption.

Priority does not override every scheduling rule. Resource requests, node
affinity, taints and tolerations, topology constraints, volumes, and Pod
readiness can still prevent scheduling.

A common CKA troubleshooting pattern:

- A high-priority Pod is Pending.
- A lower-priority Pod occupies suitable resources.
- The scheduler preempts the lower-priority Pod, if its policy allows
  preemption and no stronger scheduling constraint prevents it.
- The high-priority Pod eventually replaces it.

## Inspect priority

View PriorityClasses:

```bash
kubectl get priorityclasses
kubectl get pc
kubectl get pc -o wide
```

View effective priority on a Pod:

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.priorityClassName}'
kubectl get pod <pod-name> -o jsonpath='{.spec.priority}'
```

Example:

```bash
kubectl get pod -A -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY_CLASS:.spec.priorityClassName,PRIORITY:.spec.priority'
```

Describe and diagnose:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector involvedObject.kind=Pod
```

In Pending Pod events, look for messages about insufficient CPU, memory, or
other resources. Preemption or `preemptionPolicy: Never` only changes who may
move; it does not make a resource exist.

## Common exam patterns

### Create and assign a class

```bash
kubectl apply -f priorityclass.yaml
kubectl apply -f pod.yaml
kubectl describe pod high-priority-pod
```

### Make a class the default

```yaml
globalDefault: true
```

Also ensure no other PriorityClass remains `globalDefault: true`, unless the
cluster task specifically asks only to update the existing default.

### Prevent a class from preempting

```yaml
preemptionPolicy: Never
```

### Demonstrate priority in a Deployment

Place `priorityClassName` under:

```text
spec.template.spec.priorityClassName
```

It does not belong under `spec.template.metadata`.

### Find the effective priority

Check, in this order:

1. `spec.priorityClassName` on the Pod.
2. The referenced PriorityClass value.
3. The global default, if no class is referenced.
4. Zero, if no class and no global default are configured.

### Change priority after creation

Changing `globalDefault` affects new Pods, not existing Pods. To change a
running Pod's class:

- Edit the Pod's owning workload template.
- Roll out or recreate the workload so new Pods are admitted with the class.

## Quick reference

```bash
# Create
kubectl apply -f priorityclass.yaml

# Inspect
kubectl get priorityclasses
kubectl describe priorityclass <name>

# Delete
kubectl delete priorityclass <name>

# View a Pod's selected class
kubectl get pod <pod-name> -o jsonpath='{.spec.priorityClassName}'

# View a Pod's resolved numeric priority
kubectl get pod <pod-name> -o jsonpath='{.spec.priority}'

# View Pods with priority fields
kubectl get pods -A -o custom-columns=\
'NAMESPACE:.metadata.namespace,NAME:.metadata.name,CLASS:.spec.priorityClassName,PRIORITY:.spec.priority'
```
