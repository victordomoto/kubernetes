# How to Update a Kubernetes Cluster

> **Note:** kubeadm only supports upgrading **one minor version at a time**
> (e.g. `1.29` → `1.30`, not `1.29` → `1.31`). Confirm your target `<version>`
> respects this before starting.

---

## Control plane

```bash
# 1. Upgrade the kubeadm package
apt-mark unhold kubeadm
apt-get update && apt install kubeadm=<version>
apt-mark hold kubeadm

# 2. Check and apply the cluster upgrade
kubeadm upgrade plan
kubeadm upgrade apply <version>

# 3. Drain the node
kubectl drain control-plane --ignore-daemonsets

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt install kubelet=<version> kubectl=<version>
apt-mark hold kubelet kubectl

# 5. Reload systemd and restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 6. Uncordon the node
kubectl uncordon control-plane
```

> **Multiple control-plane nodes?** Only the *first* control-plane node uses
> `kubeadm upgrade apply <version>`. Any additional control-plane nodes
> follow the same steps as worker nodes below, using `kubeadm upgrade node`
> instead.

---

## Worker

> Run `kubectl drain` and `kubectl uncordon` from a machine with admin
> kubeconfig access (typically the control-plane node), since worker nodes
> usually don't have `kubectl`/`admin.conf` configured.

```bash
# 1. Upgrade the kubeadm package
apt-mark unhold kubeadm
apt-get update && apt install kubeadm=<version>
apt-mark hold kubeadm

# 2. Apply the node upgrade
kubeadm upgrade node

# 3. Drain the node (run from control-plane)
kubectl drain worker-1 --ignore-daemonsets

# 4. Upgrade kubelet
apt-mark unhold kubelet
apt-get update && apt install kubelet=<version>
apt-mark hold kubelet

# 5. Reload systemd and restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 6. Uncordon the node (run from control-plane)
kubectl uncordon worker-1
```