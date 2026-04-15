# Configure Topology-Aware Scheduling for GPU Workloads

Multi-node GPU training jobs are extremely sensitive to network topology. An 8-node training job where nodes communicate over InfiniBand performs orders of magnitude better when all nodes are in the same network proximity group versus spread across fault domains. This guide covers reading topology from Azure IMDS, labeling nodes, and configuring the Kubernetes scheduler for topology-aware placement.

---

## Prerequisites

- Kubernetes cluster on Azure VMSS with GPU nodes
- GPU nodes in a Proximity Placement Group (for InfiniBand workloads)
- `kubectl` cluster-admin access
- NVIDIA GPU Operator installed

---

## Step 1: Read Instance Metadata for Topology Information

Each Azure VM exposes topology information via IMDS. Run this on a GPU node to see what's available:

```bash
curl -s -H "Metadata: true" \
  "http://169.254.169.254/metadata/instance/compute?api-version=2021-12-13" | jq '{
    zone: .zone,
    location: .location,
    faultDomain: .platformFaultDomain,
    updateDomain: .platformUpdateDomain,
    vmScaleSetName: .vmScaleSetName,
    proximityPlacementGroupId: .proximityPlacementGroupId
  }'
```

Example output:

```json
{
  "zone": "2",
  "location": "eastus2",
  "faultDomain": "0",
  "updateDomain": "0",
  "vmScaleSetName": "vmss-k8s-gpu-ib",
  "proximityPlacementGroupId": "/subscriptions/.../proximityPlacementGroups/ppg-gpu-training"
}
```

---

## Step 2: Label Nodes with Topology Information

Kubernetes automatically populates some topology labels via the Azure cloud provider:

| Label | Source | Example |
|---|---|---|
| `topology.kubernetes.io/region` | IMDS `location` | `eastus2` |
| `topology.kubernetes.io/zone` | IMDS `zone` | `eastus2-2` |
| `kubernetes.io/hostname` | Node hostname | `vmss-k8s-gpu-ib_0` |

Add custom topology labels for InfiniBand and Proximity Placement Groups using a DaemonSet that reads IMDS and applies labels:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: topology-labeler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: topology-labeler
  template:
    metadata:
      labels:
        app: topology-labeler
    spec:
      hostNetwork: true
      tolerations:
        - operator: Exists
      initContainers:
        - name: labeler
          image: bitnami/kubectl:1.29
          command:
            - sh
            - -c
            - |
              IMDS="http://169.254.169.254/metadata/instance/compute?api-version=2021-12-13"
              META=$(curl -s -H "Metadata: true" "$IMDS")
              
              FAULT_DOMAIN=$(echo $META | grep -o '"platformFaultDomain":"[^"]*"' | cut -d'"' -f4)
              UPDATE_DOMAIN=$(echo $META | grep -o '"platformUpdateDomain":"[^"]*"' | cut -d'"' -f4)
              VMSS_NAME=$(echo $META | grep -o '"vmScaleSetName":"[^"]*"' | cut -d'"' -f4)
              PPG_ID=$(echo $META | grep -o '"proximityPlacementGroupId":"[^"]*"' | cut -d'"' -f4)
              PPG_NAME=$(echo $PPG_ID | rev | cut -d'/' -f1 | rev)
              
              NODE_NAME=$(cat /etc/hostname)
              
              kubectl label node "$NODE_NAME" \
                azure.com/fault-domain="$FAULT_DOMAIN" \
                azure.com/update-domain="$UPDATE_DOMAIN" \
                azure.com/vmss-name="$VMSS_NAME" \
                --overwrite
              
              if [ -n "$PPG_NAME" ]; then
                kubectl label node "$NODE_NAME" \
                  azure.com/proximity-placement-group="$PPG_NAME" \
                  --overwrite
              fi
              
              echo "Labels applied to $NODE_NAME"
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests:
              cpu: 1m
              memory: 4Mi
EOF
```

Verify labels:

```bash
kubectl get nodes -l accelerator=nvidia \
  -o custom-columns=\
NAME:.metadata.name,\
ZONE:.metadata.labels.'topology\.kubernetes\.io/zone',\
FAULT:.metadata.labels.'azure\.com/fault-domain',\
PPG:.metadata.labels.'azure\.com/proximity-placement-group'
```

---

## Step 3: Configure Pod Affinity/Anti-Affinity for Distributed Training

### Co-locate all pods in the same Proximity Placement Group

For multi-node training, all workers must be in the same PPG to get InfiniBand connectivity:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training
spec:
  completions: 8
  parallelism: 8
  completionMode: Indexed
  template:
    spec:
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  job-name: distributed-training
              topologyKey: azure.com/proximity-placement-group
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: accelerator
                    operator: In
                    values:
                      - nvidia
                  - key: ib
                    operator: In
                    values:
                      - "true"
      containers:
        - name: trainer
          image: nvcr.io/nvidia/pytorch:24.01-py3
          resources:
            limits:
              nvidia.com/gpu: 8
          env:
            - name: NCCL_DEBUG
              value: INFO
            - name: NCCL_IB_DISABLE
              value: "0"
            - name: NCCL_NET_GDR_LEVEL
              value: "2"
```

### Spread across fault domains for high availability

For inference deployments where you want HA:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: inference-server
          topologyKey: azure.com/fault-domain
```

---

## Step 4: Configure Proximity Placement Groups for InfiniBand

Create a PPG and associate the GPU VMSS with it (see also [GPU Nodes guide](gpu-nodes-vmss.md)):

```bash
export RESOURCE_GROUP="rg-k8s-vmss-prod"
export LOCATION="eastus2"

# Create PPG
az ppg create \
  --resource-group $RESOURCE_GROUP \
  --name ppg-gpu-training \
  --type Ultra \
  --location $LOCATION \
  --intent-vm-sizes Standard_ND96asr_v4

# Associate existing VMSS with PPG
PPG_ID=$(az ppg show \
  --resource-group $RESOURCE_GROUP \
  --name ppg-gpu-training \
  --query id -o tsv)

az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name vmss-k8s-gpu-ib \
  --set proximityPlacementGroup.id=$PPG_ID
```

> **Important:** Associating an existing VMSS with a PPG requires all instances to be deallocated first. For new deployments, set the PPG at creation time.

---

## Step 5: Topology Spread Constraints

Topology Spread Constraints are a first-class Kubernetes feature for distributing pods across topology domains:

```yaml
# Spread 8 training pods evenly across nodes in the same PPG zone
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        job-name: distributed-training
```

For GPU inference, spread across availability zones:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: inference-server
```

---

## Step 6: Example — 8-Node H100 Training Job with Topology-Aware Placement

This example deploys an 8-node distributed training job using PyTorch DDP, requiring:
- All 8 pods on `Standard_ND96isr_H100_v5` nodes
- All nodes in the same PPG (for InfiniBand)
- One pod per node (8 H100 GPUs per pod)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: h100-training-job
  labels:
    training-run: llm-pretrain-001
spec:
  completions: 8
  parallelism: 8
  completionMode: Indexed
  backoffLimit: 3
  template:
    metadata:
      labels:
        training-run: llm-pretrain-001
    spec:
      restartPolicy: OnFailure
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      affinity:
        # All pods must be co-located in the same PPG
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  training-run: llm-pretrain-001
              topologyKey: azure.com/proximity-placement-group
        # All pods must be on H100 nodes
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - Standard_ND96isr_H100_v5
      # One pod per node
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              training-run: llm-pretrain-001
      volumes:
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: 64Gi
        - name: training-data
          persistentVolumeClaim:
            claimName: training-dataset-pvc
      containers:
        - name: trainer
          image: nvcr.io/nvidia/pytorch:24.01-py3
          command:
            - torchrun
            - --nproc-per-node=8
            - --nnodes=8
            - --node-rank=$(JOB_COMPLETION_INDEX)
            - --master-addr=$(MASTER_ADDR)
            - --master-port=29500
            - /workspace/train.py
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
            - name: MASTER_ADDR
              value: "h100-training-job-0.h100-training-headless"
            - name: NCCL_IB_DISABLE
              value: "0"
            - name: NCCL_DEBUG
              value: "WARN"
            - name: NCCL_NET_GDR_LEVEL
              value: "2"
            - name: NCCL_TOPO_FILE
              value: "/var/run/nvidia-topologyd/virtualTopology.xml"
          resources:
            limits:
              nvidia.com/gpu: "8"
              memory: "900Gi"
              cpu: "96"
            requests:
              nvidia.com/gpu: "8"
              memory: "800Gi"
              cpu: "96"
          volumeMounts:
            - name: shm
              mountPath: /dev/shm
            - name: training-data
              mountPath: /data
---
# Headless service for pod-to-pod DNS resolution
apiVersion: v1
kind: Service
metadata:
  name: h100-training-headless
spec:
  clusterIP: None
  selector:
    training-run: llm-pretrain-001
  ports:
    - port: 29500
      name: nccl
EOF
```

Monitor the job:

```bash
kubectl get pods -l training-run=llm-pretrain-001 -o wide
kubectl logs -l training-run=llm-pretrain-001 --prefix -f
```
