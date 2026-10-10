# Kubernetes Storage for CKA

This is the core storage knowledge you need for the CKA exam: persistent volumes, volume claims, storage classes, access modes, reclaim policies, and how Kubernetes binds storage to Pods.

---

## 1. Why storage matters in Kubernetes

Containers are ephemeral. If a container restarts, its filesystem is lost. Kubernetes solves this by separating:

- the container runtime and its writable layer
- the persistent data layer provided by Kubernetes volumes

For stateful workloads, we need data that survives Pod restarts and rescheduling.

### Important distinction

- `emptyDir`: temporary storage tied to the Pod lifecycle
- `hostPath`: storage on the node filesystem, not recommended for production
- `PersistentVolume` / `PersistentVolumeClaim`: cluster-level, persistent storage
- `StorageClass`: defines how persistent storage is dynamically provisioned

---

## 2. Volumes in Pods

A Pod can mount several volume types. The most important to remember for CKA are:

### Ephemeral volumes

#### `emptyDir`

- Created when a Pod starts
- Shared across containers in the same Pod
- Deleted when the Pod is deleted
- Good for scratch space, caches, temporary data

```yaml
volumes:
- name: cache
  emptyDir: {}
```

#### `hostPath`

- Points to a filesystem path on the node
- Not portable or safe for production
- Often used only for debugging or single-node lab setups

```yaml
volumes:
- name: data
  hostPath:
    path: /var/lib/myapp
    type: DirectoryOrCreate
```

### Config and secret data

- `configMap`
- `secret`
- `projected`

These are not long-term storage for application data; they are for configuration and credentials.

---

## 3. Persistent storage model

Kubernetes uses a two-layer storage model:

- `PersistentVolume` (PV): actual storage resource provisioned by cluster/admin
- `PersistentVolumeClaim` (PVC): request by a Pod/application for storage

This decouples storage provisioning from app consumption.

### PV

A PV is a cluster resource representing storage. It has:

- `capacity`
- `accessModes`
- `persistentVolumeReclaimPolicy`
- `storageClassName`
- `mountOptions`
- `volumeMode`
- `nfs`, `hostPath`, or CSI-based `persistentVolumeSource`

Example:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/pv-data
```

### PVC

A PVC is a request for storage. Pods consume storage by mounting a PVC, not directly the PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi
```

Then the Pod mounts it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## 4. Static vs dynamic provisioning

### Static provisioning

The administrator creates the PV manually. The Pod then creates a PVC that matches it.

Use case:
- pre-created storage on NFS, iSCSI, cloud disks, etc.
- storage team manages backend details

### Dynamic provisioning

The cluster creates a PV automatically when a PVC is created, using a `StorageClass`.

This is the standard way in modern Kubernetes.

The flow is:

1. User creates PVC
2. Kubernetes finds a matching `StorageClass`
3. StorageClass's provisioner creates a backing volume
4. Kubernetes creates a PV automatically
5. PVC binds to the PV
6. Pod mounts the PVC

---

## 5. StorageClass

A `StorageClass` tells Kubernetes how to provision storage dynamically.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: WaitForFirstConsumer
```

### Key fields

#### `provisioner`

The CSI driver or in-tree plugin that creates storage.

Examples:
- `kubernetes.io/aws-ebs`
- `kubernetes.io/gce-pd`
- `csi-driver-host-path`
- cloud-specific CSI drivers

#### `reclaimPolicy`

Controls what happens to the backing volume when the PVC is deleted.

- `Retain`: keep the underlying storage, manual cleanup required
- `Delete`: delete the underlying storage automatically
- `Recycle`: deprecated and removed in newer Kubernetes versions

#### `allowVolumeExpansion`

If enabled, the size of the PV may be increased later.

#### `volumeBindingMode`

Controls when the PV is provisioned/bound.

- `Immediate`: bind as soon as the PVC exists
- `WaitForFirstConsumer`: delay binding until a Pod using the PVC is scheduled

This is especially useful for topology-aware storage, such as zone-specific disks.

---

## 6. Access modes

A PV and PVC can define access modes. These define how many nodes can mount the volume concurrently.

### `ReadWriteOnce` (RWO)

- A volume can be mounted read-write by a single node
- Most common for block storage
- Works well for many standard workloads

### `ReadOnlyMany` (ROX)

- Can be mounted read-only by multiple nodes

### `ReadWriteMany` (RWX)

- Can be mounted read-write by multiple nodes
- Used for shared storage like NFS, CephFS, GlusterFS

### `ReadWriteOncePod` (RWOP)

- A volume can be mounted by a single Pod, even if multiple Pods are on the same node
- Useful for highly isolated workloads

### Exam tip

A PVC can request one or more access modes, but the actual PV must satisfy them.

Example:

```yaml
accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
```

The storage backend must support that combination.

---

## 7. Volume modes

Two modes exist:

### Filesystem mode (default)

The volume is mounted as a filesystem.

```yaml
volumeMode: Filesystem
```

### Block mode

The volume is used as a raw block device.

```yaml
volumeMode: Block
```

Important for workloads that need direct block device access, such as databases or ISCSI-backed storage.

---

## 8. Binding and selection

A PVC selects a PV by matching the storage class, access modes, size, and sometimes labels.

Example:

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast
```

If a matching PV exists, the PVC binds to it. If no PV exists and a dynamic provisioner is configured, Kubernetes provisions one automatically.

---

## 9. StatefulSet and persistent storage

StatefulSets use persistent storage to preserve identity and data across restarts.

Each replica gets its own stable identity and often its own PVC.

Example pattern:

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast
    resources:
      requests:
        storage: 10Gi
```

This is the standard way to run databases or other stateful applications on Kubernetes.

---

## 10. Useful kubectl commands

```bash
# list storage classes
kubectl get sc

# list persistent volumes
kubectl get pv

# list persistent volume claims
kubectl get pvc

# show details
kubectl describe pvc <name>
kubectl describe pv <name>

# inspect a storage class
kubectl get storageclass <name> -o yaml

# create a PVC from a YAML file
kubectl apply -f pvc.yaml

# create a pod that uses a PVC
kubectl apply -f pod.yaml
```

### Useful checks

```bash
kubectl get pv,pvc -A
kubectl get storageclass
kubectl get events --sort-by=.metadata.creationTimestamp
```

If a PVC is stuck in `Pending`, check:

- storage class exists
- provisioner works
- access mode matches available storage
- cluster has enough backend capacity
- `WaitForFirstConsumer` may delay binding

---

## 11. Common exam scenarios

### Scenario 1: PVC remains Pending

Possible causes:

- no matching `StorageClass`
- no matching `PersistentVolume`
- wrong access mode
- provisioner failure
- no node available for `WaitForFirstConsumer`

### Scenario 2: Pod cannot mount volume

Possible causes:

- PVC not bound
- PV access mode incompatible with node constraints
- volume not available due to topology
- storage backend not reachable

### Scenario 3: Need shared storage across multiple Pods

Use:

- `ReadWriteMany` storage
- NFS, CephFS, or a CSI driver that supports RWX

### Scenario 4: Want to preserve data after PVC deletion

Set reclaim policy to `Retain`.

### Scenario 5: Need node-local placement for volumes

Use `WaitForFirstConsumer` and topology-aware storage solutions.

---

## 12. Mappings to CKA objectives

For CKA, focus on the following:

- identify when a volume is ephemeral vs persistent
- explain `PV` vs `PVC`
- bind a PVC to a PV manually or dynamically
- choose correct `accessModes`
- use `StorageClass` for dynamic provisioning
- understand `reclaimPolicy`
- know why a PVC is Pending
- know how StatefulSets use volumes

---

## 13. Quick cheat sheet

| Concept | Meaning | Exam takeaway |
|---|---|---|
| `emptyDir` | Temporary Pod storage | Lost with Pod deletion |
| `hostPath` | Node filesystem path | Not production-safe |
| `PV` | Cluster storage object | Actual storage |
| `PVC` | Storage request | Used by Pods |
| `StorageClass` | Dynamic provisioning template | Most common in labs |
| `RWO` | One node read-write | Common default |
| `ROX` | Multiple node read-only | Shared read-only storage |
| `RWX` | Multiple node read-write | Shared writable storage |
| `Retain` | Keep volume after claim delete | Manual cleanup |
| `Delete` | Delete backing volume | Automatic cleanup |
| `Immediate` | Bind right away | Simple case |
| `WaitForFirstConsumer` | Delay until Pod scheduling | Best for topology-aware storage |

---

## 14. Example: complete dynamic provisioning flow

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: web-data
```

This is a typical CKA-style storage flow: PVC -> StorageClass -> PV -> Pod mount.

---

## 15. Final takeaway

For the CKA, remember this:

- Containers have ephemeral filesystem storage.
- Kubernetes persistent storage is built around PVs and PVCs.
- StorageClasses automate provisioning.
- Access modes define mount behavior.
- Reclaim policies decide what happens when storage is no longer needed.
- Stateful apps depend on persistent volumes, especially in StatefulSets.

If you understand the relationship between `Pod` -> `PVC` -> `StorageClass`/`PV` -> backend storage, you are in strong shape for the storage portion of the exam.
