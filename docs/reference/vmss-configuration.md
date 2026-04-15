# VMSS Configuration Reference for Kubernetes

This reference covers recommended VMSS settings, supported VM SKUs, instance metadata fields, Cluster Autoscaler tags, Azure RBAC permissions, and VMSS limits relevant to self-hosted Kubernetes deployments.

---

## Recommended VMSS Settings for Kubernetes

### Orchestration Mode

Always use **Flexible** orchestration mode for new Kubernetes deployments.

```bash
az vmss create \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 1 \
  ...
```

`--platform-fault-domain-count 1` places all instances in a single fault domain, which is standard for Kubernetes worker pools that rely on Kubernetes for HA rather than Azure fault domain distribution.

### Health Probes

Configure the Application Health Extension to report node health. The VMSS uses this for automatic instance repair.

```json
{
  "extensionProfile": {
    "extensions": [
      {
        "name": "ApplicationHealthLinux",
        "properties": {
          "publisher": "Microsoft.ManagedServices",
          "type": "ApplicationHealthLinux",
          "typeHandlerVersion": "1.0",
          "settings": {
            "protocol": "tcp",
            "port": 10248,
            "requestPath": "/healthz"
          }
        }
      }
    ]
  }
}
```

Port 10248 is the kubelet health check endpoint. This ensures VMSS considers a node unhealthy only when kubelet itself is unreachable.

Via CLI:

```bash
az vmss extension set \
  --resource-group $RESOURCE_GROUP \
  --vmss-name $WORKER_VMSS_NAME \
  --name ApplicationHealthLinux \
  --publisher Microsoft.ManagedServices \
  --version 1.0 \
  --settings '{"protocol":"tcp","port":10248,"requestPath":"/healthz"}'
```

### Scale-In Policy

| Policy | Behavior | Recommended For |
|---|---|---|
| `Default` | Balance across fault/update domains | General worker pools |
| `NewestVM` | Delete most recently created VM | Spot pools (removes newest, least-settled nodes) |
| `OldestVM` | Delete oldest VM | Pools managed by manual upgrade scripts |

For Cluster Autoscaler-managed pools, CA selects the specific instance to delete — the VMSS scale-in policy only applies to Azure-native autoscale actions:

```bash
az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name $WORKER_VMSS_NAME \
  --scale-in-policy NewestVM
```

### Automatic Instance Repair

Enable automatic repair with a sufficient grace period to prevent VMSS from deleting a node before Kubernetes has had time to restart kubelet:

```bash
az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name $WORKER_VMSS_NAME \
  --enable-automatic-repairs true \
  --automatic-repairs-grace-period 30  # minutes
```

**Minimum recommended grace period: 30 minutes.** Setting it too low causes VMSS to delete nodes that are temporarily unreachable during upgrades or transient disruptions.

### Overprovisioning

Disable overprovisioning for Kubernetes. Overprovisioning creates extra VMs during scale-out and deletes them once the new instances are healthy, which confuses the Kubernetes node lifecycle:

```bash
az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name $WORKER_VMSS_NAME \
  --set singlePlacementGroup=false \
  --set overprovision=false
```

---

## Instance Metadata Fields Relevant to Kubernetes

All fields accessible at `http://169.254.169.254/metadata/instance/compute?api-version=2021-12-13`.

| Field | K8s Usage | Example Value |
|---|---|---|
| `subscriptionId` | ProviderID construction | `a1b2c3d4-...` |
| `resourceGroupName` | ProviderID, cloud provider config | `rg-k8s-vmss-prod` |
| `name` | Node hostname / ProviderID (Flexible mode) | `vmss-k8s-workers_0` |
| `vmId` | Unique VM identifier | `xxxxxxxx-xxxx-...` |
| `vmScaleSetName` | Cluster Autoscaler node group discovery | `vmss-k8s-workers` |
| `location` | `topology.kubernetes.io/region` label | `eastus2` |
| `zone` | `topology.kubernetes.io/zone` label | `2` |
| `platformFaultDomain` | Custom topology labels | `0` |
| `platformUpdateDomain` | Custom topology labels | `0` |
| `sku` | Node instance type labeling | `Standard_D8s_v5` |
| `proximityPlacementGroupId` | Topology-aware scheduling | `/subscriptions/.../ppg-gpu-training` |
| `storageProfile.osDisk.managedDisk.storageAccountType` | Storage class configuration | `Premium_LRS` |

Scheduled Events endpoint: `http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01`

| Event Type | Meaning | Handler Action |
|---|---|---|
| `Terminate` | VM will be deleted | Drain + cordon |
| `Preempt` | Spot VM eviction | Drain + cordon |
| `Reboot` | Planned maintenance reboot | Optional: drain |
| `Redeploy` | VM moving to different host | Optional: drain |
| `Freeze` | Brief pause for live migration | Usually no action needed |

---

## VMSS Tags Used by Cluster Autoscaler

### Required Discovery Tags

| Tag Key | Value | Purpose |
|---|---|---|
| `k8s.io/cluster-autoscaler/enabled` | `"true"` | Enable CA discovery |
| `k8s.io/cluster-autoscaler/<cluster-name>` | `"owned"` | Cluster ownership |
| `kubernetes.io/cluster/<cluster-name>` | `"owned"` | Cloud provider cluster membership |

### Optional Resource Hint Tags

These allow CA to predict what a new node will look like without having to query the VM for capacity:

| Tag Key | Value | Purpose |
|---|---|---|
| `k8s.io/cluster-autoscaler/node-template/resources/cpu` | `"8"` | vCPU count hint |
| `k8s.io/cluster-autoscaler/node-template/resources/memory` | `"32Gi"` | Memory hint |
| `k8s.io/cluster-autoscaler/node-template/resources/nvidia.com/gpu` | `"4"` | GPU count hint |
| `k8s.io/cluster-autoscaler/node-template/label/accelerator` | `"nvidia"` | Node label hint |
| `k8s.io/cluster-autoscaler/node-template/taint/nvidia.com/gpu` | `"present:NoSchedule"` | Taint hint |

---

## Supported VM SKUs for Kubernetes Worker Nodes

### General Purpose (Recommended for most workloads)

| SKU | vCPUs | RAM | Notes |
|---|---|---|---|
| Standard_D4s_v5 | 4 | 16 GiB | Control plane, light workers |
| Standard_D8s_v5 | 8 | 32 GiB | Standard worker nodes |
| Standard_D16s_v5 | 16 | 64 GiB | Heavier workloads |
| Standard_D32s_v5 | 32 | 128 GiB | Large pod capacity |
| Standard_D48s_v5 | 48 | 192 GiB | Large pod capacity |

### Memory Optimized

| SKU | vCPUs | RAM | Notes |
|---|---|---|---|
| Standard_E8s_v5 | 8 | 64 GiB | In-memory caches, databases |
| Standard_E16s_v5 | 16 | 128 GiB | Large in-memory workloads |
| Standard_E32s_v5 | 32 | 256 GiB | Very large in-memory workloads |

### GPU

| SKU | vCPUs | GPU | GPU Memory | Use Case |
|---|---|---|---|---|
| Standard_NC4as_T4_v3 | 4 | 1x T4 | 16 GB | Light inference |
| Standard_NC16as_T4_v3 | 16 | 4x T4 | 64 GB | Inference |
| Standard_NC24ads_A100_v4 | 24 | 1x A100 | 80 GB | Training/inference |
| Standard_NC96ads_A100_v4 | 96 | 4x A100 | 320 GB | Multi-GPU training |
| Standard_ND96asr_v4 | 96 | 8x A100 + IB | 640 GB | Distributed training |
| Standard_ND96isr_H100_v5 | 96 | 8x H100 + IB | 640 GB | Large model training |

---

## VMSS Limits

| Resource | Limit | Notes |
|---|---|---|
| Max instances per VMSS (Flexible) | 1,000 | Per VMSS resource |
| Max instances per VMSS (Uniform) | 1,000 | Per VMSS resource |
| Max VMSS per subscription (per region) | 2,500 | Soft limit, requestable |
| Max NICs per VM | Varies by SKU | D8s_v5: 4 NICs |
| Max data disks per VM | Varies by SKU | D8s_v5: 16 data disks |
| VMSS scale operation timeout | 10 minutes | Default ARM operation timeout |
| Cluster Autoscaler max node provision time | 15 minutes (configurable) | Set via `--max-node-provision-time` |

---

## Required Azure RBAC Permissions for Kubernetes Automation

### Cluster Autoscaler (Managed Identity)

Minimum permissions on the resource group containing the VMSS:

| Role | Scope | Purpose |
|---|---|---|
| `Virtual Machine Contributor` | Resource group | Create, delete, reimage VMSS instances |
| `Reader` | Resource group | List VMSS and VM resources |

Custom role (minimum permissions):

```json
{
  "Name": "Kubernetes Cluster Autoscaler",
  "Description": "Minimum permissions for K8s Cluster Autoscaler on VMSS",
  "Actions": [
    "Microsoft.Compute/virtualMachineScaleSets/read",
    "Microsoft.Compute/virtualMachineScaleSets/write",
    "Microsoft.Compute/virtualMachineScaleSets/virtualmachines/read",
    "Microsoft.Compute/virtualMachineScaleSets/virtualmachines/write",
    "Microsoft.Compute/virtualMachineScaleSets/virtualmachines/delete",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Network/networkInterfaces/read",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/<subscription-id>"]
}
```

### Azure Cloud Controller Manager

| Role | Scope | Purpose |
|---|---|---|
| `Virtual Machine Contributor` | Resource group | Node lifecycle management |
| `Network Contributor` | VNet resource group | Load balancer management |
| `Storage Blob Data Contributor` | Storage account | CSI driver (if used) |

### Node Bootstrap (kubeadm join automation)

If using automation to bootstrap nodes (custom script extension or cloud-init), the automation identity needs:

| Role | Scope | Purpose |
|---|---|---|
| `Reader` | Resource group | Read cluster metadata |
| `Key Vault Secrets User` | Key Vault | Read bootstrap tokens (if stored in KV) |
