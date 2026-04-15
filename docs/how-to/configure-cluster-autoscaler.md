# Configure the Kubernetes Cluster Autoscaler with Azure VMSS

The Kubernetes Cluster Autoscaler (CA) automatically adjusts the number of nodes in your cluster based on pod scheduling pressure and node utilization. This guide covers deploying CA with the Azure cloud provider, configuring VMSS for autoscaler discovery, and validating scale-out and scale-in behavior.

---

## Prerequisites

- A running Kubernetes cluster on Azure VMSS (see [Quickstart](../quickstart/deploy-kubeadm-vmss.md))
- `kubectl` configured with cluster-admin access
- Azure CLI authenticated (`az login`)
- Worker VMSS tagged with autoscaler discovery tags (set during VMSS creation)

---

## How the Cluster Autoscaler Works with VMSS

The Cluster Autoscaler runs as a Deployment in the `kube-system` namespace. It periodically checks for:

1. **Pending pods**: pods that cannot be scheduled due to insufficient resources → triggers scale-out
2. **Underutilized nodes**: nodes where all pods could fit on other nodes → triggers scale-in (after a configurable cool-down)

CA communicates with Azure via the ARM API to call `VMSS.Update` (scale-out) or `VMSS.DeleteInstances` (scale-in). Before deleting an instance, CA:
1. Cordons the node (`kubectl cordon`)
2. Drains pods (`kubectl drain`), respecting PodDisruptionBudgets
3. Waits for graceful termination
4. Calls the Azure API to delete the VMSS instance

---

## Step 1: Create a Managed Identity for the Autoscaler

The Cluster Autoscaler needs permission to read VMSS properties and modify instance counts.

```bash
export RESOURCE_GROUP="rg-k8s-vmss-prod"
export LOCATION="eastus2"
export CLUSTER_NAME="k8s-vmss-prod"
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export MI_NAME="mi-k8s-autoscaler"

# Create user-assigned managed identity
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $MI_NAME

export MI_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $MI_NAME \
  --query clientId -o tsv)

export MI_RESOURCE_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $MI_NAME \
  --query id -o tsv)

echo "Managed Identity Client ID: $MI_CLIENT_ID"
```

---

## Step 2: Assign RBAC Permissions

The autoscaler needs the following permissions on the resource group containing the VMSS:

```bash
export RG_RESOURCE_ID=$(az group show \
  --name $RESOURCE_GROUP \
  --query id -o tsv)

# Virtual Machine Contributor — allows VMSS scale operations
az role assignment create \
  --assignee $MI_CLIENT_ID \
  --role "Virtual Machine Contributor" \
  --scope $RG_RESOURCE_ID

# Reader — allows listing VMSS and VM resources
az role assignment create \
  --assignee $MI_CLIENT_ID \
  --role "Reader" \
  --scope $RG_RESOURCE_ID
```

If your VMSS is in a different resource group from your VNet, also grant Reader on the VNet resource group.

---

## Step 3: Assign the Managed Identity to Control Plane VMs

The Cluster Autoscaler pod runs on the control plane. Assign the managed identity to all control plane VMs so the pod can authenticate:

```bash
for i in 1 2 3; do
  az vm identity assign \
    --resource-group $RESOURCE_GROUP \
    --name "vm-controlplane-0${i}" \
    --identities $MI_RESOURCE_ID
done
```

---

## Step 4: Configure VMSS Tags for Autoscaler Discovery

The Cluster Autoscaler discovers VMSS node groups via tags. Ensure your worker VMSS has these tags:

```bash
export WORKER_VMSS_NAME="vmss-k8s-workers"

az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name $WORKER_VMSS_NAME \
  --tags \
    "k8s.io/cluster-autoscaler/enabled=true" \
    "k8s.io/cluster-autoscaler/$CLUSTER_NAME=owned" \
    "kubernetes.io/cluster/$CLUSTER_NAME=owned"
```

For node count bounds, add min/max tags (these are read by CA as hints):

```bash
az vmss update \
  --resource-group $RESOURCE_GROUP \
  --name $WORKER_VMSS_NAME \
  --tags \
    "k8s.io/cluster-autoscaler/node-template/resources/cpu=8" \
    "k8s.io/cluster-autoscaler/node-template/resources/memory=32Gi"
```

---

## Step 5: Create the Azure Cloud Provider ConfigMap

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: azure-cloud-config
  namespace: kube-system
data:
  azure.json: |
    {
      "cloud": "AzurePublicCloud",
      "tenantId": "$(az account show --query tenantId -o tsv)",
      "subscriptionId": "$SUBSCRIPTION_ID",
      "resourceGroup": "$RESOURCE_GROUP",
      "location": "$LOCATION",
      "useManagedIdentityExtension": true,
      "userAssignedIdentityID": "$MI_CLIENT_ID",
      "vmType": "vmss",
      "vmssCacheTTL": 60,
      "vmssSizeRefreshPeriod": 30
    }
EOF
```

---

## Step 6: Deploy the Cluster Autoscaler

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: cluster-autoscaler
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      containers:
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.3
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=azure
            - --cloud-config=/etc/kubernetes/azure.json
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=label:k8s.io/cluster-autoscaler/enabled=true,k8s.io/cluster-autoscaler/$CLUSTER_NAME=owned
            - --balance-similar-node-groups
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --scale-down-utilization-threshold=0.5
            - --max-node-provision-time=15m
          volumeMounts:
            - name: azure-cloud-config
              mountPath: /etc/kubernetes/azure.json
              subPath: azure.json
              readOnly: true
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
      volumes:
        - name: azure-cloud-config
          configMap:
            name: azure-cloud-config
        - name: ssl-certs
          hostPath:
            path: /etc/ssl/certs/ca-certificates.crt
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: [events, endpoints]
    verbs: [create, patch]
  - apiGroups: [""]
    resources: [pods/eviction]
    verbs: [create]
  - apiGroups: [""]
    resources: [pods/status]
    verbs: [update]
  - apiGroups: [""]
    resources: [endpoints]
    resourceNames: [cluster-autoscaler]
    verbs: [get, update]
  - apiGroups: [""]
    resources: [nodes]
    verbs: [watch, list, get, update]
  - apiGroups: [""]
    resources: [namespaces, pods, services, replicationcontrollers, persistentvolumeclaims, persistentvolumes]
    verbs: [watch, list, get]
  - apiGroups: [extensions]
    resources: [replicasets, daemonsets]
    verbs: [watch, list, get]
  - apiGroups: [policy]
    resources: [poddisruptionbudgets]
    verbs: [watch, list]
  - apiGroups: [apps]
    resources: [statefulsets, replicasets, daemonsets]
    verbs: [watch, list, get]
  - apiGroups: [storage.k8s.io]
    resources: [storageclasses, csinodes, csidrivers, csistoragecapacities]
    verbs: [watch, list, get]
  - apiGroups: [batch, extensions]
    resources: [jobs]
    verbs: [get, list, watch, patch]
  - apiGroups: [coordination.k8s.io]
    resources: [leases]
    verbs: [create]
  - apiGroups: [coordination.k8s.io]
    resources: [leases]
    resourceNames: [cluster-autoscaler]
    verbs: [get, update]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system
EOF
```

---

## Step 7: Verify Deployment

```bash
kubectl -n kube-system get pods -l app=cluster-autoscaler
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=50
```

Look for log lines like:

```
I0115 12:00:00.000000 cluster_autoscaler] Starting main loop
I0115 12:00:01.000000 azure_manager] Found VMSS: vmss-k8s-workers (min: 1, max: 10)
```

---

## Step 8: Configure Min/Max Node Counts

Min and max are configured via CA command-line flags. The `--node-group-auto-discovery` flag discovers groups; min/max override the VMSS capacity bounds.

To explicitly set bounds per VMSS, use the `--nodes` flag instead of auto-discovery:

```bash
# Format: min:max:vmss-resource-id
--nodes=1:20:/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Compute/virtualMachineScaleSets/$WORKER_VMSS_NAME
```

Update the Deployment to use this flag:

```bash
kubectl -n kube-system patch deployment cluster-autoscaler \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/command/-","value":"--nodes=1:20:/subscriptions/'"$SUBSCRIPTION_ID"'/resourceGroups/'"$RESOURCE_GROUP"'/providers/Microsoft.Compute/virtualMachineScaleSets/'"$WORKER_VMSS_NAME"'"}]'
```

---

## Step 9: Test Scale-Out

Deploy a workload that requires more resources than currently available:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ca-test-scaleout
spec:
  replicas: 20
  selector:
    matchLabels:
      app: ca-test
  template:
    metadata:
      labels:
        app: ca-test
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
EOF

# Watch CA add nodes
kubectl get nodes -w &
kubectl -n kube-system logs -l app=cluster-autoscaler -f
```

After scale-out completes, clean up:

```bash
kubectl delete deployment ca-test-scaleout
```

Scale-in will begin after `--scale-down-unneeded-time` (default 10 minutes).

---

## Troubleshooting Autoscaler Issues

### CA is not scaling out

1. Check for pending pods:
   ```bash
   kubectl get pods --all-namespaces --field-selector status.phase=Pending
   ```
2. Check CA logs for "no candidates" or tag mismatch:
   ```bash
   kubectl -n kube-system logs -l app=cluster-autoscaler | grep -i "no node group"
   ```
3. Verify VMSS tags are correct:
   ```bash
   az vmss show --resource-group $RESOURCE_GROUP --name $WORKER_VMSS_NAME --query tags
   ```

### CA is not scaling in

1. Check if nodes have non-evictable pods (DaemonSets, pods with local storage):
   ```bash
   kubectl -n kube-system logs -l app=cluster-autoscaler | grep -i "scale down"
   ```
2. Verify PodDisruptionBudgets are not blocking drain:
   ```bash
   kubectl get pdb --all-namespaces
   ```
3. Check utilization threshold (`--scale-down-utilization-threshold`). Default is 0.5 (50%).

### Permission errors

```bash
# Verify role assignment
az role assignment list --assignee $MI_CLIENT_ID --output table

# Test ARM access from a control plane node
curl -s -H "Metadata: true" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
```
