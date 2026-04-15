# Handle Spot VM Evictions Gracefully in Self-Hosted Kubernetes

Azure Spot VMs can reduce worker node costs by 60–90% compared to on-demand pricing. The trade-off is that Azure can reclaim Spot VMs with a 30-second eviction notice. This guide shows you how to detect eviction signals and drain Kubernetes nodes before the VM is terminated.

---

## Prerequisites

- A running Kubernetes cluster on Azure VMSS
- `kubectl` with cluster-admin access
- Azure CLI authenticated
- At least one VMSS configured with Spot priority (see Step 1)

---

## Understanding Azure Spot VM Eviction Behavior

When Azure needs to reclaim a Spot VM, it:

1. Sends an **IMDS Scheduled Event** of type `Preempt` or `Terminate` to the VM
2. Sets a `NotBefore` time 30 seconds in the future
3. Terminates the VM at `NotBefore`

The 30-second window is non-negotiable. Your handler must cordon and drain the node within this window.

Poll the Scheduled Events endpoint from within the VM:

```bash
curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01"
```

Eviction event example:

```json
{
  "DocumentIncarnation": 1,
  "Events": [
    {
      "EventId": "a1b2c3d4-e5f6-...",
      "EventType": "Preempt",
      "ResourceType": "VirtualMachine",
      "Resources": ["vmss-workers-spot_0"],
      "EventStatus": "Scheduled",
      "NotBefore": "2024-01-15T12:00:30Z",
      "Description": "Virtual machine is being preempted.",
      "EventSource": "Platform"
    }
  ]
}
```

---

## Step 1: Create a Spot Worker VMSS

```bash
export RESOURCE_GROUP="rg-k8s-vmss-prod"
export LOCATION="eastus2"
export CLUSTER_NAME="k8s-vmss-prod"
export SPOT_VMSS_NAME="vmss-k8s-workers-spot"

WORKER_SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name vnet-k8s-prod \
  --name snet-workers \
  --query id -o tsv)

az vmss create \
  --resource-group $RESOURCE_GROUP \
  --name $SPOT_VMSS_NAME \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 1 \
  --vm-sku Standard_D8s_v5 \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --subnet $WORKER_SUBNET_ID \
  --instance-count 0 \
  --priority Spot \
  --eviction-policy Delete \
  --max-price -1 \
  --os-disk-size-gb 128 \
  --storage-sku Premium_LRS \
  --tags \
    "k8s.io/cluster-autoscaler/enabled=true" \
    "k8s.io/cluster-autoscaler/$CLUSTER_NAME=owned" \
    "kubernetes.io/cluster/$CLUSTER_NAME=owned" \
    "spot=true"
```

> **Note:** `--eviction-policy Delete` is required for Spot VMs in VMSS. `Deallocate` is not supported for Spot.
> `--max-price -1` means "pay up to the on-demand price" (recommended — avoids unnecessary evictions from price spikes).

---

## Step 2: Deploy the Spot Interruption Handler DaemonSet

This DaemonSet runs a polling container on every Spot node. When a `Preempt` event is detected, it cordons and drains the node.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spot-interruption-handler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: spot-interruption-handler
rules:
  - apiGroups: [""]
    resources: [nodes]
    verbs: [get, list, patch, update]
  - apiGroups: [""]
    resources: [pods]
    verbs: [list, get]
  - apiGroups: [""]
    resources: [pods/eviction]
    verbs: [create]
  - apiGroups: [apps]
    resources: [daemonsets]
    verbs: [get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spot-interruption-handler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: spot-interruption-handler
subjects:
  - kind: ServiceAccount
    name: spot-interruption-handler
    namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: spot-interruption-handler
  namespace: kube-system
  labels:
    app: spot-interruption-handler
spec:
  selector:
    matchLabels:
      app: spot-interruption-handler
  template:
    metadata:
      labels:
        app: spot-interruption-handler
    spec:
      serviceAccountName: spot-interruption-handler
      hostNetwork: true
      tolerations:
        - operator: Exists
      nodeSelector:
        spot: "true"
      priorityClassName: system-node-critical
      containers:
        - name: handler
          image: python:3.11-slim
          command:
            - python3
            - -c
            - |
              import json, os, subprocess, time, urllib.request
              
              NODE_NAME = os.environ.get("NODE_NAME", "")
              IMDS_URL = "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01"
              POLL_INTERVAL = 5  # seconds
              
              def get_scheduled_events():
                  req = urllib.request.Request(IMDS_URL, headers={"Metadata": "true"})
                  try:
                      with urllib.request.urlopen(req, timeout=2) as resp:
                          return json.loads(resp.read())
                  except Exception as e:
                      print(f"IMDS poll failed: {e}")
                      return {"Events": []}
              
              def drain_node(node_name):
                  print(f"Eviction detected for {node_name}. Cordoning...")
                  subprocess.run(["kubectl", "cordon", node_name], check=False)
                  print("Draining node...")
                  subprocess.run([
                      "kubectl", "drain", node_name,
                      "--ignore-daemonsets",
                      "--delete-emptydir-data",
                      "--force",
                      "--grace-period=20",
                      "--timeout=25s"
                  ], check=False)
                  print("Drain complete.")
              
              print(f"Spot interruption handler started. Watching node: {NODE_NAME}")
              while True:
                  events = get_scheduled_events()
                  for event in events.get("Events", []):
                      if event.get("EventType") in ("Preempt", "Terminate"):
                          resources = event.get("Resources", [])
                          if any(NODE_NAME in r for r in resources):
                              drain_node(NODE_NAME)
                              time.sleep(60)  # avoid re-triggering
                  time.sleep(POLL_INTERVAL)
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
EOF
```

---

## Step 3: Configure Graceful Termination Periods

Kubernetes must be told to give pods enough time to finish before being killed. The 30-second eviction window is tight — configure your workloads accordingly.

**Set `terminationGracePeriodSeconds` on workloads running on Spot nodes:**

```yaml
spec:
  terminationGracePeriodSeconds: 25  # Must be < 30 seconds for Spot
  containers:
    - name: my-app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
```

**Configure `kubelet` graceful shutdown on worker nodes** (add to `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` or `/var/lib/kubelet/config.yaml`):

```yaml
# /var/lib/kubelet/config.yaml
shutdownGracePeriod: 25s
shutdownGracePeriodCriticalPods: 10s
```

Apply and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

## Step 4: Taint Spot Nodes

Taint Spot nodes so that only pods that explicitly tolerate Spot are scheduled there:

```bash
# Apply taint to existing Spot nodes
kubectl taint nodes -l spot=true \
  kubernetes.azure.com/scalesetpriority=spot:NoSchedule

# For new nodes, add to the VMSS custom script extension or cloud-init
# kubeadm join ... after which:
# kubectl taint node <node-name> kubernetes.azure.com/scalesetpriority=spot:NoSchedule
```

Add the toleration to workloads that should run on Spot:

```yaml
tolerations:
  - key: kubernetes.azure.com/scalesetpriority
    operator: Equal
    value: spot
    effect: NoSchedule
```

---

## Step 5: Configure Mixed Spot + On-Demand with Cluster Autoscaler

Run control plane and critical system workloads on On-Demand nodes. Run batch and training workloads on Spot.

Configure CA with two node groups:

```bash
# In the CA deployment command flags, add both VMSS:
--nodes=1:10:/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Compute/virtualMachineScaleSets/vmss-k8s-workers
--nodes=0:20:/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Compute/virtualMachineScaleSets/vmss-k8s-workers-spot
```

CA will prefer the cheapest (Spot) node group when scaling out pods that tolerate Spot.

---

## Best Practices for Training Workloads on Spot

### Use checkpointing

Training jobs should checkpoint model state every N steps so that a Spot eviction only loses the work since the last checkpoint:

```yaml
# Example: PyTorch training job with checkpoint volume
volumes:
  - name: checkpoint-volume
    persistentVolumeClaim:
      claimName: training-checkpoints
containers:
  - name: trainer
    volumeMounts:
      - name: checkpoint-volume
        mountPath: /checkpoints
    env:
      - name: CHECKPOINT_DIR
        value: /checkpoints
      - name: CHECKPOINT_INTERVAL_STEPS
        value: "100"
```

### Use Kubernetes Job restart policy

```yaml
apiVersion: batch/v1
kind: Job
spec:
  backoffLimit: 10  # allow up to 10 restarts
  template:
    spec:
      restartPolicy: OnFailure
      tolerations:
        - key: kubernetes.azure.com/scalesetpriority
          operator: Equal
          value: spot
          effect: NoSchedule
```

### Avoid running stateful services on Spot

Never run etcd, databases, or persistent message queues on Spot VMs. These require stable storage and cannot tolerate sudden termination.

### Monitor Spot eviction rates

```bash
# Check Azure Spot eviction history via Azure Monitor (requires diagnostic settings)
az monitor metrics list \
  --resource "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Compute/virtualMachineScaleSets/$SPOT_VMSS_NAME" \
  --metric "Evicted VMs" \
  --interval PT1H
```
