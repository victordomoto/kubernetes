Role Based Access Control is how kubernetes controls who can do what in the cluster.

* Who? -> user, group or service account
* Can do what? -> get, list, create, delete, etc
* On which resources? -> pods, deployments, secrets, etc
* Where? -> namespace or cluster-wide  


* ClusterRoles / roles : define permissions
* ClusterRoleBinding roleBiding: assign permissions
* Subjects: who receives permissions
* Rules: what actions are allowed


# ClusterRole

A cluster role define permissions at cluster level. 
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```
## When to use ClusterRole?
* When you need access across namespaces
* You work with cluster resources (nodes, pv, etc)


# ClusterRoleBinding
assign a ClusterRole to a subject.
Binds:
* User
* Group
* ServiceAccount
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods-global
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

# ServiceAccount
Is an identity used by pods.

* Used by applications running inside the cluster
* Automatically mounted inside pods
* Used for API authentications

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: default
```

How it works in pratice:
1. Pod uses a ServiceAccount
2. ServiceAccount is bound to a role
3. Pod get permissions from that role
```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: nginx
```

```
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa
kubectl auth can-i delete pods --as=system:serviceaccount:default:my-sa
```
