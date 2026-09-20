# Kubernetes Network Policies: CKA Cheat Sheet

---

## 1. What they are

- A NetworkPolicy is a **L3/L4 firewall for pods** (IP, port, protocol). No L7 (HTTP paths, headers).
- Namespaced resource, `networking.k8s.io/v1`.
- **Enforced by the CNI**, not by Kubernetes itself. If the CNI doesn't support it, the object is accepted and silently ignored.
  - Enforce: Calico, Cilium, Weave, Antrea
  - Do NOT enforce: plain Flannel
- Default behavior: **all pods can talk to all pods** (non-isolated).

---

## 2. Core rules (memorize these)

1. A pod is **isolated** for a direction (ingress/egress) only when at least one policy selects it AND lists that direction in `policyTypes`.
2. Once isolated, **only traffic explicitly allowed** is permitted. There are **no deny rules**, only allow rules.
3. Policies are **additive** (union). Multiple policies on one pod = all their allows combined.
4. `podSelector: {}` = **all pods** in the namespace.
5. Policies only affect **pods in their own namespace** (the target). Peers can come from other namespaces via `namespaceSelector`.
6. Reply traffic of an allowed connection is allowed automatically (stateful).
7. Traffic to/from the pod's own node (kubelet probes) is always allowed.

---

## 3. Anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example
  namespace: prod
spec:
  podSelector:              # WHO this policy applies to (target pods)
    matchLabels:
      app: db
  policyTypes: [Ingress, Egress]
  ingress:                  # who can reach the target
  - from:
    - podSelector: {matchLabels: {app: api}}
    ports:
    - protocol: TCP
      port: 5432
  egress:                   # where the target can go
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
        except: [10.1.0.0/16]
    ports:
    - protocol: TCP
      port: 443
```

Peer selectors (`from` / `to`):

| Selector | Matches |
|---|---|
| `podSelector` | Pods by label, **in the policy's namespace** (unless combined with namespaceSelector) |
| `namespaceSelector` | **All pods** in namespaces matching the label |
| `ipBlock` | CIDR ranges (usually for external IPs; pod IPs are not reliable here) |

---

## 4. AND vs OR (the classic trap)

**AND**: one list item with both selectors. Pods labeled `app=frontend` in namespace `web`:

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: web}
    podSelector:                       # no leading dash = AND
      matchLabels: {app: frontend}
```

**OR**: two list items. Any pod in `web`, OR pods `app=frontend` in the policy's own namespace:

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: web}
  - podSelector:                       # leading dash = OR
      matchLabels: {app: frontend}
```

Same logic between `from` items and `ports`: separate `-` entries are ORed.

---

## 5. Essential patterns

### Default deny ingress (namespace-wide)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod
spec:
  podSelector: {}
  policyTypes: [Ingress]
```

### Default deny egress
```yaml
spec:
  podSelector: {}
  policyTypes: [Egress]
```

### Default deny all (ingress + egress)
```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

### Allow all ingress (explicit)
```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - {}
```

### Allow DNS egress (required after egress deny)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: prod
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: kube-system}
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
```

### Allow from ingress controller namespace
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: ingress-nginx}
  ports:
  - {protocol: TCP, port: 8080}
```

### Port range
```yaml
ports:
- protocol: TCP
  port: 32000
  endPort: 32768
```

---

## 6. When to use them

- **Zero-trust baseline**: default-deny per namespace, then open only what's needed.
- **Multi-tenant clusters**: isolate namespaces from each other.
- **Protect databases/backends**: only the app tier can reach them.
- **Restrict egress**: block pods from reaching the internet or metadata endpoints (e.g. `169.254.169.254`).
- **Compliance** (PCI, etc.): document and enforce allowed flows.

Not for: L7 rules, auth, encryption, or logging. Use a service mesh or CNI-specific policies (Cilium L7, Calico GlobalNetworkPolicy) for those.

---

## 7. Commands

```bash
# List / inspect
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>
kubectl get netpol <name> -n <ns> -o yaml

# Check labels used by selectors
kubectl get pods -n <ns> --show-labels
kubectl get ns --show-labels

# Label a namespace (if selector needs it)
kubectl label ns web team=frontend

# Test connectivity
kubectl run test --rm -it --image=busybox -n web --restart=Never -- nc -zv db.prod 5432
kubectl run test --rm -it --image=busybox -n web --restart=Never -- wget -qO- --timeout=2 http://svc.prod
kubectl exec -n web <pod> -- nc -zv <pod-ip> 5432

# Which CNI is running?
kubectl get pods -n kube-system -o wide | grep -Ei 'calico|cilium|flannel|weave|antrea'
```

Note: `kubectl create` has **no** netpol generator. Copy a template from the docs (search "network policy" on kubernetes.io) and edit it.

---

## 8. Debugging checklist

1. **CNI supports netpol?** If not, nothing you write matters.
2. **Does a policy select the pod?** `describe netpol` and compare `podSelector` with `--show-labels`.
3. **Right `policyTypes`?** Omitting `Egress` means egress is untouched.
4. **AND/OR dash placement** correct?
5. **Namespace labels exist?** `namespaceSelector` needs real labels. Since 1.21 every namespace has `kubernetes.io/metadata.name`.
6. **Egress deny broke DNS?** Add the DNS allow rule.
7. **Both sides covered?** Pod A egress to B needs A's egress allow AND B's ingress allow (if both are isolated).
8. **Service vs pod**: policies match **pod** IPs/ports (`targetPort`), not Service ports.
9. **Test from the correct namespace and with the correct labels.**

---

## 9. Exam tips

- Read the question for **namespace** and **labels**. Wrong namespace = zero points.
- Use the docs page (kubernetes.io > Network Policies) and adapt the YAML; don't write from scratch.
- Check **existing** policies first (`get netpol -A`); some tasks say "pick the least permissive policy from these YAMLs".
- Verify with a test pod after applying.
- Named ports work (`port: http`) if the container declares `name: http`.
- Quick reference: no `deny` keyword, no `kubectl create networkpolicy`, `{}` means "all", empty `ingress: []` or no `ingress` key with policyType Ingress means "deny all".

---

## 10. Quick reference table

| Goal | `podSelector` | `policyTypes` | Rules |
|---|---|---|---|
| Deny all in ns | `{}` | Ingress, Egress | none |
| Deny only ingress | `{}` | Ingress | none |
| Allow from app X | target pods | Ingress | `from: podSelector` |
| Allow from other ns | target pods | Ingress | `from: namespaceSelector` |
| Allow from ns AND label | target pods | Ingress | one item with both selectors |
| Allow to external CIDR | source pods | Egress | `to: ipBlock` |
| Allow DNS | `{}` | Egress | `to: kube-system` port 53 UDP/TCP |