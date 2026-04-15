# Add GPU Worker Nodes to a Self-Hosted Kubernetes Cluster on VMSS

This guide covers adding GPU-enabled VMSS node pools to an existing self-hosted Kubernetes cluster, installing the NVIDIA GPU Operator, configuring GPUDirect RDMA for multi-node training, and running GPU workloads.

---

## Prerequisites

- Running Kubernetes cluster on Azure VMSS (see [Quickstart](../quickstart/deploy-kubeadm-vmss.md))
- `kubectl` with cluster-admin access
- `helm` 3.x installed locally
- Azure subscription with quota for GPU VM SKUs (request via Azure portal if needed)

---

## Step 1: Create a GPU VMSS Node Pool

Azure GPU VM families for Kubernetes:

| Series | GPU | Use Case |
|---|---|---|
| NC A100 v4 | NVIDIA A100 80GB | Training, inference |
| ND A100 v4 | NVIDIA A100 80GB + InfiniBand | Multi-node distributed training |
| ND H100 v5 | NVIDIA H100 80GB + InfiniBand | Large model training |
| NC T4 v3 | NVIDIA T4 | Inference, lighter training |

```bash
export RESOURCE_GROUP="rg-k8s-vmss-prod"
export CLUSTER_NAME="k8s-vmss-prod"
export GPU_VMSS_NAME="vmss-k8s-gpu"
export GPU_VM_SIZE="Standard_NC96ads_A100_v4"  # 96 vCPUs, 4x A100 80GB

WORKER_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name vnet-k8s-prod \
  --name snet-workers \
  --query id -o tsv)

az vmss create \
  --resource-group $RESOURCE_GROUP \
  --name $GPU_VMSS_NAME \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 1 \
  --vm-sku $GPU_VM_SIZE \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --subnet $WORKER_SUBNET_ID \
  --instance-count 1 \
  --os-disk-size-gb 256 \
  --data-disk-sizes-gb 1024 \
  --storage-sku Premium_LRS \
  --accelerated-networking true \
  --tags \
    "k8s.io/cluster-autoscaler/enabled=true" \
    "k8s.io/cluster-autoscaler/$CLUSTER_NAME=owned" \
    "kubernetes.io/cluster/$CLUSTER_NAME=owned" \
    "gpu=true"
```

---

## Step 2: Bootstrap GPU Nodes and Join the Cluster

SSH into each GPU node and run the standard Kubernetes node setup from the [Quickstart Step 6](../quickstart/deploy-kubeadm-vmss.md#step-6-bootstrap-control-plane-nodes).

**Do not pre-install NVIDIA drivers manually.** The GPU Operator will manage driver installation.

Join the cluster:

```bash
# On control plane — generate a new join token
kubeadm token create --print-join-command

# On GPU node — run the output from the above command
sudo kubeadm join <lb-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Label GPU nodes:

```bash
kubectl label nodes -l gpu=true \
  accelerator=nvidia \
  node-role.kubernetes.io/gpu-worker=""

# Taint so only GPU workloads are scheduled here
kubectl taint nodes -l gpu=true \
  nvidia.com/gpu=present:NoSchedule
```

---

## Step 3: Install the NVIDIA GPU Operator

The GPU Operator automates installation of NVIDIA drivers, CUDA, the container toolkit, and the device plugin.

```bash
# Add NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# Create namespace
kubectl create namespace gpu-operator

# Install GPU Operator
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --set driver.enabled=true \
  --set driver.version="535.161.07" \
  --set toolkit.enabled=true \
  --set devicePlugin.enabled=true \
  --set dcgmExporter.enabled=true \
  --set migManager.enabled=false \
  --set tolerations[0].key=nvidia.com/gpu \
  --set tolerations[0].operator=Exists \
  --set tolerations[0].effect=NoSchedule \
  --wait
```

Wait for GPU Operator components:

```bash
kubectl -n gpu-operator get pods -w
```

All pods should reach `Running` or `Completed` state within 5–10 minutes.

---

## Step 4: Verify GPU Availability

```bash
# Check the nvidia.com/gpu resource is reported
kubectl get nodes -l accelerator=nvidia -o custom-columns=\
NAME:.metadata.name,\
GPU:.status.allocatable.'nvidia\.com/gpu',\
DRIVER:.metadata.labels.'nvidia\.com/cuda\.driver\.rev'

# Run a test pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
  containers:
    - name: cuda-test
      image: nvidia/cuda:12.3.1-base-ubuntu22.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
EOF

kubectl wait --for=condition=Complete pod/gpu-test --timeout=120s
kubectl logs gpu-test
kubectl delete pod gpu-test
```

Expected output includes GPU model, driver version, and CUDA version.

---

## Step 5: Pin CUDA Version

To prevent the GPU Operator from upgrading CUDA automatically, pin the driver version in the Helm values:

```bash
helm upgrade gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --reuse-values \
  --set driver.version="535.161.07"
```

To verify the installed driver version across nodes:

```bash
kubectl get nodes -l accelerator=nvidia \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.nvidia\.com/cuda\.driver\.rev}{"\n"}{end}'
```

---

## Step 6: Enable GPUDirect RDMA for Multi-Node Training

GPUDirect RDMA allows GPU memory to be transferred directly over InfiniBand without going through the CPU. This requires:
- ND or NDv2/v4/v5 series VMs (InfiniBand-capable)
- InfiniBand driver (OFED)
- A Proximity Placement Group (for low-latency connectivity)

### Create InfiniBand-capable GPU VMSS

```bash
export GPU_IB_VMSS_NAME="vmss-k8s-gpu-ib"
export GPU_IB_VM_SIZE="Standard_ND96asr_v4"  # ND A100 v4 with IB

# Create a Proximity Placement Group
az ppg create \
  --resource-group $RESOURCE_GROUP \
  --name ppg-gpu-training \
  --type Ultra \
  --location $LOCATION

PPG_ID=$(az ppg show \
  --resource-group $RESOURCE_GROUP \
  --name ppg-gpu-training \
  --query id -o tsv)

az vmss create \
  --resource-group $RESOURCE_GROUP \
  --name $GPU_IB_VMSS_NAME \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 1 \
  --vm-sku $GPU_IB_VM_SIZE \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --subnet $WORKER_SUBNET_ID \
  --instance-count 0 \
  --os-disk-size-gb 256 \
  --ppg $PPG_ID \
  --storage-sku Premium_LRS \
  --tags \
    "k8s.io/cluster-autoscaler/$CLUSTER_NAME=owned" \
    "kubernetes.io/cluster/$CLUSTER_NAME=owned" \
    "gpu=true" "ib=true"
```

### Install OFED and GPUDirect via GPU Operator

```bash
helm upgrade gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --reuse-values \
  --set driver.rdma.enabled=true \
  --set driver.rdma.useHostMofed=false
```

The GPU Operator will install the MOFED (Mellanox OFED) driver on InfiniBand nodes automatically.

### Install the Network Operator (for Multus + SR-IOV)

```bash
helm repo add network-operator https://helm.ngc.nvidia.com/nvidia/network-operator
helm repo update

helm install network-operator network-operator/network-operator \
  --namespace network-operator \
  --create-namespace \
  --set deploySR-IOV=true \
  --wait
```

Verify InfiniBand devices are available:

```bash
# On a GPU+IB node
ibstat
# Should show: State: Active, Physical state: LinkUp
```

---

## Step 7: Configure MIG (Multi-Instance GPU) Partitioning

MIG allows a single A100 or H100 GPU to be partitioned into multiple isolated instances.

Enable MIG on specific nodes:

```bash
kubectl label node <gpu-node-name> nvidia.com/mig.config=all-3g.40gb

# Monitor MIG configuration
kubectl -n gpu-operator logs -l app=mig-manager -f
```

Available MIG profiles for A100 80GB:

| Profile | GPU Memory | Compute Slices |
|---|---|---|
| `1g.10gb` | 10 GB | 1 |
| `2g.20gb` | 20 GB | 2 |
| `3g.40gb` | 40 GB | 3 |
| `4g.40gb` | 40 GB | 4 |
| `7g.80gb` | 80 GB | 7 (full GPU) |

After MIG is configured, the node will advertise `nvidia.com/mig-3g.40gb` resources instead of `nvidia.com/gpu`.

---

## Troubleshooting GPU Node Issues

### `nvidia.com/gpu` not showing in node resources

1. Check GPU Operator pods:
   ```bash
   kubectl -n gpu-operator get pods
   kubectl -n gpu-operator describe pod <device-plugin-pod>
   ```
2. Verify the node has the correct taint tolerated by GPU Operator:
   ```bash
   kubectl get node <node> -o jsonpath='{.spec.taints}'
   ```
3. Check NVIDIA driver installation:
   ```bash
   # SSH into GPU node
   nvidia-smi
   # If not found, check gpu-operator driver pod logs
   kubectl -n gpu-operator logs -l app=nvidia-driver-daemonset
   ```

### GPU workload stuck in Pending

```bash
kubectl describe pod <pod-name>
# Look for: "0/N nodes are available: N Insufficient nvidia.com/gpu"
```

Verify the pod requests are correct:
```yaml
resources:
  limits:
    nvidia.com/gpu: 1  # must be in limits, not just requests
```

### GPUDirect RDMA performance is poor

1. Verify all nodes are in the same Proximity Placement Group
2. Check InfiniBand link state: `ibstat` should show `Active`
3. Verify NCCL is using IB: set `NCCL_DEBUG=INFO` and look for `IB` in NCCL output
4. Ensure `rdma-core` is installed: `dpkg -l rdma-core`
