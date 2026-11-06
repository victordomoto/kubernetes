# Pod Security Admission (PSA)

Pod Security Admission is Kubernetes' built-in mechanism for enforcing
Pod Security Standards. It evaluates Pods when they are admitted and applies
the policy configured on their namespace. For the CKA, know the three policy
levels, the three enforcement modes, how to label a namespace, and how to
diagnose a rejected Pod.

## What PSA does

- PSA is a built-in admission controller; it is not a separate resource, CRD,
  or add-on to install.
- It has been stable since Kubernetes v1.25.
- It applies Pod Security Standards to Pods in a namespace.
- The policy is selected with labels on the **Namespace**, not on the Pod.
- It checks Pods at admission. Existing Pods are not evicted when a namespace's
  policy changes.
- `enforce` applies to Pods, not workload objects. A Deployment can be accepted
  while the Pods created from its template are rejected. `warn` and `audit`
  also check workload templates, which can reveal violations earlier.
- PSA replaced PodSecurityPolicies, which were removed in Kubernetes v1.25.

## The three policy levels

| Level | Meaning | Typical use |
|---|---|---|
| `privileged` | Unrestricted; no meaningful restrictions from the Pod Security Standard | Trusted system workloads |
| `baseline` | Blocks known privilege escalations while allowing common workload patterns | General-purpose workloads |
| `restricted` | Strong hardening based on current Pod security best practices | Workloads that can meet stricter requirements |

The levels are cumulative in strictness: `privileged` is least restrictive,
`baseline` is stricter, and `restricted` is most restrictive.

Examples of settings that can violate stricter levels include privileged
containers, host namespaces, host ports, `hostPath` volumes, unsafe Linux
capabilities, running as root, privilege escalation, or missing seccomp
settings. The exact checks are versioned; consult the Pod Security Standards
reference for a specific Kubernetes version rather than memorizing every
individual rule.

## The three modes

Each mode evaluates a selected level independently:

| Mode | Result for a violating Pod |
|---|---|
| `enforce` | Rejects Pod admission |
| `audit` | Admits the Pod and records a policy violation in audit metadata |
| `warn` | Admits the Pod and returns a warning to the requestor |

Use all three labels to enforce one level while observing a stricter level
before switching production workloads:

```text
pod-security.kubernetes.io/enforce
pod-security.kubernetes.io/audit
pod-security.kubernetes.io/warn
```

Each label's value is a level: `privileged`, `baseline`, or `restricted`.
For example, enforce `baseline` but warn when a Pod would fail `restricted`.

## Configure a namespace

Apply labels to the namespace. `--overwrite` is useful when changing an
existing policy:

```bash
# See current namespace labels
kubectl get namespace team-a --show-labels

# Enforce baseline; report violations against restricted as warnings
kubectl label --overwrite namespace team-a \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

# Also record restricted violations in audit metadata
kubectl label --overwrite namespace team-a \
  pod-security.kubernetes.io/audit=restricted

# Enforce restricted
kubectl label --overwrite namespace team-a \
  pod-security.kubernetes.io/enforce=restricted

# Remove an enforcement label
kubectl label namespace team-a pod-security.kubernetes.io/enforce-
```

If a mode label is absent, PSA uses the cluster's configured defaults. In a
standard cluster without customized admission configuration, the default level
is `privileged`, so an unlabeled namespace is not automatically restricted.
Do not assume a namespace has a policy—inspect its labels.

### Pin the policy version

Policy checks evolve with Kubernetes. A version label pins a mode to the
Pod Security Standards for a specific Kubernetes minor version:

```bash
kubectl label --overwrite namespace team-a \
  pod-security.kubernetes.io/enforce-version=v1.30
```

Set the version to the Kubernetes minor version required by the task; `latest`
is also a valid value. Pinning avoids policy changes when upgrading Kubernetes.
Version labels are available for each mode, for example
`pod-security.kubernetes.io/warn-version`. If the task does not request a
version pin, inspect the cluster's existing labels and follow the task's
instructions.

## Restricted-compatible Pod example

Restricted workloads generally need to avoid root and privilege escalation,
drop Linux capabilities, and set an allowed seccomp profile. Confirm the
application image works with the chosen user and filesystem settings; passing
admission does not guarantee that the application will run successfully.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-example
  namespace: team-a
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

`readOnlyRootFilesystem: true` is useful hardening, but it is not by itself a
general Restricted-level requirement. Other requirements can depend on the
Pod's fields and the policy version.

## Troubleshooting and verification

```bash
# Inspect policy labels
kubectl get namespace team-a --show-labels
kubectl describe namespace team-a

# Preview how enforcement would affect existing Pods in the namespace
kubectl label --dry-run=server --overwrite namespace team-a \
  pod-security.kubernetes.io/enforce=restricted

# Inspect a failed workload and its Pods
kubectl describe deployment app -n team-a
kubectl get replicasets,pods -n team-a
kubectl describe pod <pod-name> -n team-a
kubectl get events -n team-a --sort-by=.metadata.creationTimestamp

# Test a Pod manifest against server-side admission without persisting it
kubectl apply --dry-run=server -f pod.yaml
```

The namespace-label dry run reports existing Pod violations without changing
the namespace. It is a useful preflight when raising an `enforce` level.

Common admission errors identify the namespace, policy level, and fields that
violated the policy. Fix the Pod template's security context or use the level
the task requests; do not weaken the namespace policy just to make a Pod start.

## CKA quick checklist

1. Read the required namespace and level in the task.
2. Check the namespace and current PSA labels before changing anything.
3. Use namespace labels—there is no `PodSecurityPolicy` object to create.
4. Choose the exact mode: `enforce` blocks, `warn` informs, and `audit` records.
5. Add or update labels with `kubectl label --overwrite namespace ...`.
6. For a strict policy, inspect the Pod's security context and volume types.
7. Verify the resulting Pods and events; an accepted Deployment does not prove
   its Pods were admitted.
8. Remember that changing a policy does not evict already-running Pods.

## Quick reference

| Requirement | Namespace label |
|---|---|
| Reject Pods that violate Baseline | `pod-security.kubernetes.io/enforce=baseline` |
| Warn about violations of Restricted | `pod-security.kubernetes.io/warn=restricted` |
| Audit violations of Restricted | `pod-security.kubernetes.io/audit=restricted` |
| Pin enforcement checks to a Kubernetes minor version | `pod-security.kubernetes.io/enforce-version=v1.30` |

Official references:

- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
