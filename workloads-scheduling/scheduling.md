# nodeSelector vs nodeAffinity

Both are ways to constrain which nodes a Pod can be scheduled on, based on node labels. `nodeAffinity` is a more expressive superset of what `nodeSelector` does.

---

## 1. nodeSelector

The simplest scheduling constraint: a flat key-value match against node labels. All keys must match (implicit AND). No operators, no soft preferences.

### When to use it
- You need a simple, hard constraint (e.g. "must run on SSD nodes").
- You don't need OR logic, operators, or fallback behavior.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-nodeselector
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: nginx
```

This Pod only schedules on nodes labeled `disktype=ssd`. If none exist, it stays `Pending`.

### How to label a node

```bash
kubectl label nodes <node-name> disktype=ssd
kubectl get nodes --show-labels
```

---

## 2. nodeAffinity

A richer version of the same concept, supporting operators and two modes: hard requirement and soft preference.

### Operators supported
`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`

### Two rule types

| Type | Behavior |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard rule — must match, or Pod won't schedule |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft rule — scheduler tries to honor it (weighted), but schedules elsewhere if it can't |

`IgnoredDuringExecution` means: if a node's labels change after the Pod is already running, the Pod is **not** evicted. The rule is only evaluated at scheduling time.

### Example: hard requirement

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-required-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: app
    image: nginx
```

### Example: soft preference with weight

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-preferred-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
      - weight: 20
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1b
  containers:
  - name: app
    image: nginx
```

The scheduler prefers `us-east-1a` nodes, but will still place the Pod on any available node if none match.

### Example: combining hard + soft

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-combined-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: app
    image: nginx
```

Must be an SSD node (hard rule); among SSD nodes, prefer `us-east-1a`.

---

## 3. Key differences

| | nodeSelector | nodeAffinity |
|---|---|---|
| Syntax | Flat key-value map | Structured expressions under `spec.affinity.nodeAffinity` |
| Operators | Equality only | `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` |
| Soft preferences | Not supported | Supported via `preferred...` + weight |
| Multiple conditions | Implicit AND across keys | AND within a `matchExpressions` block, OR across `nodeSelectorTerms` |
| Use case | Simple, hard match | Complex rules, "prefer but don't require" |

---

## 4. Useful commands

```bash
# label a node
kubectl label nodes <node-name> zone=us-east-1a

# check node labels
kubectl get nodes --show-labels

# check why a Pod is stuck Pending (common cause: no node matches required affinity/selector)
kubectl describe pod <pod-name>

# remove a label
kubectl label nodes <node-name> disktype-
```

---

## 5. Related topic: taints and tolerations

`nodeSelector`/`nodeAffinity` work from the Pod's side — the Pod chooses which nodes it wants. **Taints and tolerations** work from the node's side — the node repels Pods unless the Pod explicitly tolerates the taint. The two mechanisms are often combined:

- Taints/tolerations: keep general workloads *off* specialized nodes.
- Node affinity: attract the *right* workloads *onto* those nodes.