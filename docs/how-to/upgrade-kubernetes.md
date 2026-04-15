# Upgrade Kubernetes on VMSS Without Downtime

This guide covers upgrading a self-hosted Kubernetes cluster deployed on Azure VMSS. The process upgrades the control plane first, then rolls worker nodes through a cordon-drain-upgrade-uncordon cycle using VMSS model updates.

---

## Prerequisites

- Kubernetes cluster running on Azure VMSS
- `kubectl` and `kubeadm` installed on control plane nodes
- Azure CLI authenticated
- Cluster running with at least 3 worker nodes (to maintain workload availability during rolling upgrade)
- etcd backup completed before starting

---

## Understanding the Upgrade Process

Kubernetes supports upgrading one minor version at a time (e.g., 1.28 → 1.29, not 1.28 → 1.30). The upgrade sequence is:

1. **Upgrade control plane** — one node at a time, starting with the primary
2. **Upgrade worker nodes** — cordon, drain, upgrade packages, rejoin
3. **Verify cluster health** at each step

For VMSS worker nodes, you have two options:
- **In-place upgrade**: SSH into each instance, upgrade packages, restart kubelet
- **Node replacement** (recommended): Update the VMSS model with a new image, then reimaged instances

This guide covers both approaches.

---

## Step 1: Back Up etcd

Always back up etcd before upgrading. Run on any control plane node:

```bash
# Get etcd pod name
ETCD_POD=$(kubectl -n kube-system get pods -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# Run etcd snapshot
kubectl -n kube-system exec $ETCD_POD -- \
  etcdctl snapshot save /var/lib/etcd/snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Copy snapshot off the node
scp azureuser@<cp1-ip>:/var/lib/etcd/snapshot-*.db ./etcd-backup/
```

---

## Step 2: Upgrade the First Control Plane Node

SSH into `vm-controlplane-01`.

```bash
# Determine available upgrade versions
sudo kubeadm upgrade plan

# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.30.1-1.1
sudo apt-mark hold kubeadm

# Verify
kubeadm version

# Apply the upgrade
sudo kubeadm upgrade apply v1.30.1 --yes

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.1-1.1 kubectl=1.30.1-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Verify from your local machine:

```bash
kubectl get nodes
# vm-controlplane-01 should show v1.30.1
# Other nodes will still show previous version — this is expected
```

---

## Step 3: Upgrade Remaining Control Plane Nodes

For `vm-controlplane-02` and `vm-controlplane-03`, the process is slightly different — use `upgrade node` instead of `upgrade apply`:

```bash
# Cordon from local machine
kubectl cordon vm-controlplane-02

# SSH into vm-controlplane-02
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubeadm=1.30.1-1.1

sudo kubeadm upgrade node

sudo apt-get install -y kubelet=1.30.1-1.1 kubectl=1.30.1-1.1
sudo apt-mark hold kubeadm kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon from local machine
kubectl uncordon vm-controlplane-02
```

Repeat for `vm-controlplane-03`.

Verify all control plane nodes are on the new version:

```bash
kubectl get nodes -l node-role.kubernetes.io/control-plane
```

---

## Step 4: Upgrade Worker Nodes — In-Place Method

For each worker node in the VMSS, cordon, drain, upgrade, and uncordon:

```bash
# List worker nodes
kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o wide

# For each worker node:
WORKER_NODE="vmss-k8s-workers-xxxxx"

# Cordon
kubectl cordon $WORKER_NODE

# Drain (wait for pods to migrate)
kubectl drain $WORKER_NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30 \
  --timeout=120s

# SSH into the worker node and upgrade
# (Get worker IP: kubectl get node $WORKER_NODE -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
WORKER_IP=$(kubectl get node $WORKER_NODE -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
ssh azureuser@$WORKER_IP "
  sudo apt-mark unhold kubeadm kubelet kubectl &&
  sudo apt-get update &&
  sudo apt-get install -y kubeadm=1.30.1-1.1 &&
  sudo kubeadm upgrade node &&
  sudo apt-get install -y kubelet=1.30.1-1.1 kubectl=1.30.1-1.1 &&
  sudo apt-mark hold kubeadm kubelet kubectl &&
  sudo systemctl daemon-reload &&
  sudo systemctl restart kubelet
"

# Uncordon
kubectl uncordon $WORKER_NODE

# Verify node version before proceeding
kubectl get node $WORKER_NODE

# Wait for node to be Ready
kubectl wait --for=condition=Ready node/$WORKER_NODE --timeout=120s
```

Repeat for each worker node. Use a rolling approach — upgrade one node at a time to maintain capacity.

---

## Step 5: Upgrade Worker Nodes — VMSS Node Replacement Method (Recommended)

This approach replaces nodes with fresh instances running a new OS image that includes the new Kubernetes packages. It's cleaner and avoids package state drift.

### Update the VMSS model

Create a new custom image or use cloud-init to install the new Kubernetes version:

```bash
# Update the VMSS to use a new image or custom script that installs K8s 1.30
az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name vmss-k8s-workers \
  --set virtualMachineProfile.extensionProfile.extensions[0].properties.settings.commandToExecute="$(cat worker-setup-1.30.sh)"
```

### Roll instances one at a time

```bash
# Get all instance IDs
INSTANCE_IDS=$(az vmss list-instances \
  --resource-group $RESOURCE_GROUP \
  --name vmss-k8s-workers \
  --query "[].instanceId" -o tsv)

for INSTANCE_ID in $INSTANCE_IDS; do
  # Get the node name for this instance
  INSTANCE_NAME=$(az vmss get-instance-view \
    --resource-group $RESOURCE_GROUP \
    --name vmss-k8s-workers \
    --instance-id $INSTANCE_ID \
    --query computerName -o tsv 2>/dev/null || \
    az vmss list-instances \
      --resource-group $RESOURCE_GROUP \
      --name vmss-k8s-workers \
      --query "[?instanceId=='$INSTANCE_ID'].osProfile.computerName" -o tsv)

  echo "Processing instance $INSTANCE_ID ($INSTANCE_NAME)..."

  # Cordon and drain
  kubectl cordon $INSTANCE_NAME
  kubectl drain $INSTANCE_NAME \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=30 \
    --timeout=180s

  # Reimage instance (applies new VMSS model)
  az vmss reimage \
    --resource-group $RESOURCE_GROUP \
    --name vmss-k8s-workers \
    --instance-ids $INSTANCE_ID

  # Wait for instance to come back
  sleep 60
  kubectl wait --for=condition=Ready node/$INSTANCE_NAME --timeout=180s
  kubectl uncordon $INSTANCE_NAME

  echo "Instance $INSTANCE_ID upgraded successfully."
done
```

---

## Step 6: Verify Cluster Health Post-Upgrade

```bash
# All nodes should be Ready and on the new version
kubectl get nodes -o wide

# Check system pods
kubectl -n kube-system get pods

# Run a test workload
kubectl create deployment upgrade-test --image=nginx:latest --replicas=3
kubectl rollout status deployment/upgrade-test
kubectl delete deployment upgrade-test

# Verify etcd health
ETCD_POD=$(kubectl -n kube-system get pods -l component=etcd -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system exec $ETCD_POD -- \
  etcdctl endpoint health --cluster \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Rollback Procedure

### Control plane rollback

Kubernetes does not support downgrading `kubeadm upgrade`. To roll back the control plane, you must restore from the etcd backup taken in Step 1.

```bash
# Stop all control plane components on all CP nodes
sudo systemctl stop kubelet

# Restore etcd snapshot (on all CP nodes)
sudo mv /var/lib/etcd /var/lib/etcd.bak
sudo mkdir -p /var/lib/etcd

ETCD_VERSION="v3.5.10"
sudo ETCDCTL_API=3 etcdctl snapshot restore /path/to/snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=vm-controlplane-01 \
  --initial-cluster="vm-controlplane-01=https://<cp1-ip>:2380,vm-controlplane-02=https://<cp2-ip>:2380,vm-controlplane-03=https://<cp3-ip>:2380" \
  --initial-cluster-token=etcd-cluster \
  --initial-advertise-peer-urls=https://<this-node-ip>:2380

sudo chown -R etcd:etcd /var/lib/etcd
sudo systemctl start kubelet
```

### Worker node rollback

For in-place upgrades, downgrade packages on each worker:

```bash
sudo apt-get install -y kubelet=1.29.3-1.1 kubeadm=1.29.3-1.1 kubectl=1.29.3-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

For VMSS node replacement, revert the VMSS model to the previous image and reimage instances.
