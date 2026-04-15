# Quickstart: Deploy a Kubernetes Cluster on Azure VMSS Using kubeadm

This quickstart walks you through deploying a highly available Kubernetes cluster on Azure using:
- A 3-node control plane backed by an Azure Load Balancer
- A worker node pool using VMSS with Flexible orchestration
- Calico as the CNI plugin

**Estimated time:** 45–60 minutes

---

## Prerequisites

- Azure subscription with Contributor or Owner access
- Azure CLI 2.55.0 or later (`az upgrade` to update)
- `kubectl` 1.28 or later installed locally
- SSH key pair (`~/.ssh/id_rsa` / `~/.ssh/id_rsa.pub`)
- Bash shell (Linux or macOS; WSL2 on Windows)

Verify Azure CLI version:

```bash
az version
az login
```

---

## Step 1: Set Variables

Set these variables at the start of your session. All subsequent commands reference them.

```bash
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export RESOURCE_GROUP="rg-k8s-vmss-prod"
export LOCATION="eastus2"
export CLUSTER_NAME="k8s-vmss-prod"
export VNET_NAME="vnet-k8s-prod"
export SUBNET_CONTROL="snet-controlplane"
export SUBNET_WORKER="snet-workers"
export NSG_NAME="nsg-k8s-prod"
export LB_NAME="lb-k8s-controlplane"
export LB_IP_NAME="pip-k8s-controlplane"
export CONTROL_VM_SIZE="Standard_D4s_v5"
export WORKER_VM_SIZE="Standard_D8s_v5"
export K8S_VERSION="1.29.3"
export ADMIN_USER="azureuser"
export SSH_KEY_PATH="$HOME/.ssh/id_rsa.pub"
export WORKER_VMSS_NAME="vmss-k8s-workers"
```

---

## Step 2: Create Resource Group and Network

```bash
# Resource group
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION

# Virtual network with two subnets
az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --address-prefix 10.0.0.0/16 \
  --subnet-name $SUBNET_CONTROL \
  --subnet-prefix 10.0.1.0/24

az network vnet subnet create \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_WORKER \
  --address-prefix 10.0.2.0/24
```

---

## Step 3: Create Network Security Group

```bash
az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name $NSG_NAME

# SSH inbound (restrict source in production)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name allow-ssh \
  --priority 100 \
  --protocol Tcp \
  --destination-port-range 22 \
  --access Allow

# Kubernetes API server
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name allow-k8s-apiserver \
  --priority 110 \
  --protocol Tcp \
  --destination-port-range 6443 \
  --access Allow

# etcd (control plane internal only)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name allow-etcd \
  --priority 120 \
  --protocol Tcp \
  --destination-port-range 2379-2380 \
  --source-address-prefix 10.0.1.0/24 \
  --access Allow

# kubelet API (control plane to workers)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name allow-kubelet \
  --priority 130 \
  --protocol Tcp \
  --destination-port-range 10250 \
  --access Allow

# Calico VXLAN
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name allow-calico-vxlan \
  --priority 140 \
  --protocol Udp \
  --destination-port-range 4789 \
  --access Allow

# NodePort range
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $NSG_NAME \
  --name allow-nodeports \
  --priority 150 \
  --protocol Tcp \
  --destination-port-range 30000-32767 \
  --access Allow

# Attach NSG to both subnets
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_CONTROL \
  --network-security-group $NSG_NAME

az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_WORKER \
  --network-security-group $NSG_NAME
```

---

## Step 4: Create the Control Plane Load Balancer

```bash
# Public IP for the load balancer (use internal LB for private clusters)
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name $LB_IP_NAME \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

export LB_PUBLIC_IP=$(az network public-ip show \
  --resource-group $RESOURCE_GROUP \
  --name $LB_IP_NAME \
  --query ipAddress -o tsv)

echo "Control plane LB IP: $LB_PUBLIC_IP"

# Load balancer
az network lb create \
  --resource-group $RESOURCE_GROUP \
  --name $LB_NAME \
  --sku Standard \
  --public-ip-address $LB_IP_NAME \
  --frontend-ip-name fe-k8s-apiserver \
  --backend-pool-name be-k8s-controlplane

# Health probe on port 6443
az network lb probe create \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME \
  --name probe-apiserver \
  --protocol Tcp \
  --port 6443 \
  --interval 5 \
  --threshold 2

# Load balancing rule for API server
az network lb rule create \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME \
  --name rule-apiserver \
  --protocol Tcp \
  --frontend-port 6443 \
  --backend-port 6443 \
  --frontend-ip-name fe-k8s-apiserver \
  --backend-pool-name be-k8s-controlplane \
  --probe-name probe-apiserver \
  --load-distribution SourceIP
```

---

## Step 5: Create Control Plane VMs

Create 3 VMs for the control plane. Each gets a NIC attached to the LB backend pool.

```bash
CONTROL_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_CONTROL \
  --query id -o tsv)

LB_BACKEND_ID=$(az network lb address-pool show \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME \
  --name be-k8s-controlplane \
  --query id -o tsv)

for i in 1 2 3; do
  # Create NIC
  az network nic create \
    --resource-group $RESOURCE_GROUP \
    --name "nic-controlplane-0${i}" \
    --vnet-name $VNET_NAME \
    --subnet $SUBNET_CONTROL \
    --lb-address-pools $LB_BACKEND_ID

  # Create VM
  az vm create \
    --resource-group $RESOURCE_GROUP \
    --name "vm-controlplane-0${i}" \
    --size $CONTROL_VM_SIZE \
    --image Ubuntu2204 \
    --admin-username $ADMIN_USER \
    --ssh-key-values $SSH_KEY_PATH \
    --nics "nic-controlplane-0${i}" \
    --os-disk-size-gb 128 \
    --storage-sku Premium_LRS \
    --no-wait
done

# Wait for all VMs to be ready
az vm wait \
  --resource-group $RESOURCE_GROUP \
  --ids $(az vm list -g $RESOURCE_GROUP --query "[?contains(name,'controlplane')].id" -o tsv) \
  --created
```

---

## Step 6: Bootstrap Control Plane Nodes

SSH into each control plane node and run this setup script. Start with `vm-controlplane-01`.

```bash
# Get the private IP of the first control plane VM
CP1_IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name vm-controlplane-01 \
  --show-details \
  --query privateIps -o tsv)

ssh $ADMIN_USER@$CP1_IP
```

Run the following **on each control plane node**:

```bash
# Disable swap (required by Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.29.3-1.1 kubeadm=1.29.3-1.1 kubectl=1.29.3-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 7: Initialize the First Control Plane Node

Run **only on `vm-controlplane-01`**:

```bash
# Replace with your actual LB public IP
LB_PUBLIC_IP="<your-lb-ip>"

sudo kubeadm init \
  --control-plane-endpoint "$LB_PUBLIC_IP:6443" \
  --upload-certs \
  --pod-network-cidr 192.168.0.0/16 \
  --kubernetes-version 1.29.3 \
  --apiserver-advertise-address $(hostname -I | awk '{print $1}')
```

Save the output — it contains the join commands for additional control plane nodes and workers.

Configure `kubectl` on this node:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 8: Install Calico CNI

**On `vm-controlplane-01`**:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

Wait for Calico to be ready:

```bash
kubectl wait --for=condition=Ready pods --all -n calico-system --timeout=120s
```

---

## Step 9: Join Remaining Control Plane Nodes

From the `kubeadm init` output, run the control plane join command on `vm-controlplane-02` and `vm-controlplane-03`:

```bash
# Example — use the actual command from your kubeadm init output
sudo kubeadm join $LB_PUBLIC_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key> \
  --apiserver-advertise-address $(hostname -I | awk '{print $1}')
```

---

## Step 10: Create the Worker Node VMSS

Back on your local machine:

```bash
WORKER_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name $SUBNET_WORKER \
  --query id -o tsv)

> **Note:** Azure CLI 2.85.0 has a known bug where `az vmss create` with Flexible orchestration returns a JSON parse error (`Extra data: line 1 column 4`). Use the ARM REST API approach below, or upgrade to CLI 2.86.0+ when available.

```bash
# Option A: ARM REST API (works with CLI 2.85.0)
PUBKEY=$(cat $SSH_KEY_PATH)
python3 -c "
import json, sys
data = {
  'location': '$LOCATION',
  'sku': {'name': '$WORKER_VM_SIZE', 'capacity': 3},
  'tags': {
    'k8s.io/cluster-autoscaler/enabled': 'true',
    'k8s.io/cluster-autoscaler/$CLUSTER_NAME': 'owned',
    'kubernetes.io/cluster/$CLUSTER_NAME': 'owned'
  },
  'properties': {
    'orchestrationMode': 'Flexible',
    'platformFaultDomainCount': 1,
    'virtualMachineProfile': {
      'storageProfile': {
        'imageReference': {'publisher': 'Canonical', 'offer': '0001-com-ubuntu-server-jammy', 'sku': '22_04-lts', 'version': 'latest'},
        'osDisk': {'createOption': 'FromImage', 'diskSizeGB': 128, 'managedDisk': {'storageAccountType': 'Premium_LRS'}}
      },
      'osProfile': {
        'computerNamePrefix': 'worker',
        'adminUsername': '$ADMIN_USER',
        'linuxConfiguration': {
          'disablePasswordAuthentication': True,
          'ssh': {'publicKeys': [{'path': '/home/$ADMIN_USER/.ssh/authorized_keys', 'keyData': sys.argv[1]}]}
        }
      },
      'networkProfile': {
        'networkApiVersion': '2020-11-01',
        'networkInterfaceConfigurations': [{'name': 'nic', 'properties': {'primary': True, 'ipConfigurations': [{'name': 'ip', 'properties': {'subnet': {'id': sys.argv[2]}}}]}}]
      }
    }
  }
}
print(json.dumps(data))
" "$PUBKEY" "$WORKER_SUBNET_ID" > /tmp/vmss-body.json

az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Compute/virtualMachineScaleSets/$WORKER_VMSS_NAME?api-version=2024-03-01" \
  --body @/tmp/vmss-body.json \
  --query "properties.provisioningState" -o tsv
```

```bash
# Option B: az vmss create (CLI 2.86.0+)
az vmss create \
  --resource-group $RESOURCE_GROUP \
  --name $WORKER_VMSS_NAME \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 1 \
  --vm-sku $WORKER_VM_SIZE \
  --image Ubuntu2204 \
  --admin-username $ADMIN_USER \
  --ssh-key-values $SSH_KEY_PATH \
  --subnet $WORKER_SUBNET_ID \
  --instance-count 3 \
  --os-disk-size-gb 128 \
  --storage-sku Premium_LRS \
  --tags \
    "k8s.io/cluster-autoscaler/enabled=true" \
    "k8s.io/cluster-autoscaler/$CLUSTER_NAME=owned" \
    "kubernetes.io/cluster/$CLUSTER_NAME=owned"
```

---

## Step 11: Bootstrap Worker Nodes and Join the Cluster

SSH into each worker node (via the control plane as a bastion, or using the worker NIC public IP if configured) and run the same [node setup script](#step-6-bootstrap-control-plane-nodes) from Step 6.

Then join the cluster using the worker join command from the `kubeadm init` output:

```bash
sudo kubeadm join $LB_PUBLIC_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token has expired (tokens expire after 24 hours), generate a new one:

```bash
# On any control plane node
kubeadm token create --print-join-command
```

---

## Step 12: Verify the Cluster

From `vm-controlplane-01` (or copy `admin.conf` to your local machine):

```bash
# All nodes should be Ready
kubectl get nodes -o wide

# System pods should all be Running
kubectl get pods -n kube-system

# Verify etcd health
kubectl -n kube-system exec -it etcd-vm-controlplane-01 -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster

# Deploy a test workload
kubectl create deployment nginx-test --image=nginx:latest --replicas=3
kubectl get pods -o wide
kubectl delete deployment nginx-test
```

Expected output:

```
NAME                  STATUS   ROLES           AGE   VERSION
vm-controlplane-01    Ready    control-plane   10m   v1.29.3
vm-controlplane-02    Ready    control-plane   8m    v1.29.3
vm-controlplane-03    Ready    control-plane   7m    v1.29.3
vmss-workers-xxxxx    Ready    <none>          5m    v1.29.3
vmss-workers-yyyyy    Ready    <none>          5m    v1.29.3
vmss-workers-zzzzz    Ready    <none>          5m    v1.29.3
```

---

## Next Steps

- [Configure Cluster Autoscaler](../how-to/configure-cluster-autoscaler.md) to automatically scale worker nodes
- [Handle Spot VM Evictions](../how-to/handle-spot-evictions.md) to reduce worker node costs
- [Add GPU Worker Nodes](../how-to/gpu-nodes-vmss.md) for ML/AI workloads
