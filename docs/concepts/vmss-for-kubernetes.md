# How VMSS Powers Self-Hosted Kubernetes

Azure Virtual Machine Scale Sets (VMSS) is the foundation that makes scalable, self-hosted Kubernetes clusters practical on Azure. This article explains what VMSS contributes to a Kubernetes deployment, how to configure it correctly, and how Kubernetes components interact with VMSS APIs and instance metadata.

---

## What VMSS Provides

### Fleet Management

VMSS manages a group of identical VMs as a single resource. Instead of provisioning and tracking individual VMs, you define a *model* — an OS image, VM size, network profile, and extension configuration — and VMSS ensures all instances conform to it. This makes Kubernetes worker node pools easy to reason about: every node in a VMSS pool runs the same image and configuration.

Key fleet management capabilities:

- **Uniform OS image**: all nodes boot from the same image version
- **Model-based updates**: changing the VMSS model triggers rolling upgrades across instances
- **Instance protection**: specific instances can be protected from scale-in (useful for nodes running critical workloads)
- **Health-based auto-repair**: unhealthy instances are automatically replaced

### Autoscaling

VMSS supports two autoscaling mechanisms:

1. **Azure Autoscale (native)** — scales based on Azure Monitor metrics (CPU, memory, queue depth). This is independent of Kubernetes and does not understand pod scheduling pressure.
2. **Kubernetes Cluster Autoscaler** — scales based on pending pods and node utilization. This is the correct mechanism for K8s workloads. It calls Azure VMSS APIs to add or remove instances.

For Kubernetes, always use the Cluster Autoscaler. Do not enable both simultaneously on the same VMSS.

### Health Repair

VMSS Automatic Instance Repair continuously monitors instance health using either:
- **Azure Load Balancer health probes**
- **Application Health Extension** (recommended for Kubernetes)

When an instance is deemed unhealthy for a configurable grace period (default: 10 minutes), VMSS deletes and recreates it. See [Troubleshooting](../troubleshooting/common-issues.md) for guidance on preventing this from conflicting with Kubernetes node management.

---

## Flexible vs. Uniform Orchestration

VMSS supports two orchestration modes:

| Feature | Flexible | Uniform |
|---|---|---|
| Instance creation API | Standard VM API | VMSS-specific API |
| Node identity (ProviderID) | `/subscriptions/.../virtualMachines/<id>` | `/subscriptions/.../virtualMachineScaleSets/<name>/virtualMachines/<id>` |
| Mix VM sizes | ✅ | ❌ |
| Spot + On-Demand mix | ✅ | ❌ |
| Ultra Disk support | ✅ | ❌ |
| Max instances per VMSS | 1,000 | 1,000 |
| Azure Cluster Autoscaler support | ✅ | ✅ |
| Recommended for K8s | ✅ **Yes** | ⚠️ Legacy |

**Use Flexible orchestration for all new Kubernetes deployments.** Uniform orchestration is a legacy mode. Flexible mode gives you standard VM semantics, easier debugging, and better feature parity with newer Azure VM capabilities.

To create a VMSS with Flexible orchestration:

```bash
az vmss create \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 1 \
  ...
```

> **Note:** `--platform-fault-domain-count 1` is required for Flexible mode when you want single-placement-group behavior typical of Kubernetes worker pools. Use a higher value if you need cross-fault-domain spreading.

---

## VMSS Instance Metadata Service (IMDS)

Every Azure VM has access to the Instance Metadata Service at `http://169.254.169.254/metadata/instance`. This non-routable address is accessible only from within the VM.

Kubernetes components read IMDS to:

| Component | IMDS Field | Purpose |
|---|---|---|
| `kubelet` | `compute.subscriptionId`, `compute.resourceGroupName`, `compute.name` | Construct the node's `ProviderID` |
| `kubelet` | `compute.location`, `compute.zone` | Populate `topology.kubernetes.io/region` and `topology.kubernetes.io/zone` labels |
| `cloud-controller-manager` | `compute.vmId`, `compute.vmScaleSetName` | Node lifecycle management |
| Cluster Autoscaler | `compute.vmScaleSetName` | Identify which VMSS the node belongs to |
| Spot handler | `scheduledevents` | Detect upcoming evictions |

Example IMDS query from a node:

```bash
curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/instance?api-version=2021-12-13" | jq .
```

The `ProviderID` format for Flexible-mode VMSS nodes is:

```
azure:///subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm-name>
```

For Uniform-mode:

```
azure:///subscriptions/<subscription-id>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachineScaleSets/<vmss-name>/virtualMachines/<instance-id>
```

This distinction matters for the Azure cloud provider configuration (`--cloud-provider=azure`) and for the Cluster Autoscaler.

---

## How the Cluster Autoscaler Integrates with VMSS

The Kubernetes Cluster Autoscaler (CA) uses the Azure cloud provider plugin to scale VMSS capacity. The integration works as follows:

1. **Discovery**: CA identifies VMSS node pools via tags. The required tags are:
   - `k8s.io/cluster-autoscaler/enabled: "true"`
   - `k8s.io/cluster-autoscaler/<cluster-name>: "owned"`

2. **Scale-out**: When pods are pending due to insufficient resources, CA calls `az vmss scale` (or equivalent ARM API) to increase the instance count.

3. **Scale-in**: When nodes are underutilized, CA selects candidates, cordons them, drains pods (respecting PodDisruptionBudgets), then calls the VMSS API to delete the specific instance.

4. **Node group bounds**: CA respects `minCount` and `maxCount` values configured in the autoscaler deployment. These map directly to VMSS minimum and maximum capacity.

The CA communicates with Azure via a Managed Identity or Service Principal bound to the VMSS. See [Configure Cluster Autoscaler](../how-to/configure-cluster-autoscaler.md) for full setup.

---

## VMSS Scale-In Policy and Kubernetes Node Drain

The VMSS scale-in policy controls *which* instance is deleted first. The default policy is `Default` (balanced across fault and update domains). For Kubernetes, this is usually acceptable.

However, CA does not use the VMSS scale-in policy — it explicitly selects which node to delete after draining. To prevent VMSS from independently deleting a node that Kubernetes is still using, set the scale-in policy to `NewestVM` or use Instance Protection:

```bash
# Protect a specific instance from scale-in (CA removes this protection before deleting)
az vmss update-instances \
  --resource-group myResourceGroup \
  --name myWorkerVMSS \
  --instance-ids 3 \
  --protect-from-scale-in true
```

For CA-managed clusters, the CA handles its own drain-then-delete logic. Disable Azure Autoscale on CA-managed VMSS to prevent race conditions.

---

## Scheduled Events and Spot VM Eviction Handling

Azure Spot VMs receive a 30-second eviction notice through the IMDS Scheduled Events endpoint before the VM is terminated. Kubernetes does not natively handle this — you must deploy a handler.

The Scheduled Events endpoint:

```bash
curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01" | jq .
```

A pending eviction looks like:

```json
{
  "Events": [
    {
      "EventId": "...",
      "EventType": "Preempt",
      "ResourceType": "VirtualMachine",
      "Resources": ["myvm"],
      "EventStatus": "Scheduled",
      "NotBefore": "2024-01-15T12:00:00Z"
    }
  ]
}
```

When a `Preempt` or `Terminate` event is detected, your handler should:
1. Cordon the node (mark unschedulable)
2. Drain running pods with `kubectl drain`
3. Optionally acknowledge the event to accelerate the eviction

See [Handle Spot VM Evictions](../how-to/handle-spot-evictions.md) for a complete DaemonSet implementation.
