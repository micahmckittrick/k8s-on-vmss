# Troubleshooting: Common Issues with Self-Hosted Kubernetes on VMSS

This guide covers the most common issues encountered when running self-hosted Kubernetes on Azure VMSS. Each issue includes the symptom, root cause, and resolution steps.

---

## 1. Nodes Not Joining the Cluster

### Symptom

`kubeadm join` hangs or returns an error. New VMSS instances start but never appear in `kubectl get nodes`.

### Root Cause A: ProviderID Format Mismatch

The Azure cloud provider expects a specific `ProviderID` format. If the cloud provider config does not match the actual VMSS orchestration mode, node registration fails.

**Flexible mode ProviderID:**
```
azure:///subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm-name>
```

**Uniform mode ProviderID:**
```
azure:///subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachineScaleSets/<vmss>/virtualMachines/<id>
```

**Fix:**

Verify your cloud provider config in `/etc/kubernetes/cloud-config` has `"vmType": "vmss"` (not `"standard"`) and that `"orchestrationMode"` matches your VMSS. Restart the kubelet after updating:

```bash
sudo systemctl restart kubelet
kubectl describe node <node-name> | grep ProviderID
```

### Root Cause B: NSG Rules Blocking kubelet Port

The API server must reach the kubelet (port 10250) on worker nodes for log streaming and exec.

**Fix:**

```bash
# Check NSG rules
az network nsg show \
  --resource-group $RESOURCE_GROUP \
  --name nsg-k8s-prod \
  --query "securityRules[?destinationPortRange=='10250']"

# Add rule if missing
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name nsg-k8s-prod \
  --name allow-kubelet-api \
  --priority 130 \
  --protocol Tcp \
  --destination-port-range 10250 \
  --source-address-prefix 10.0.1.0/24 \
  --access Allow
```

### Root Cause C: DNS Resolution Failure for API Server

**Fix:**

On the joining node, verify it can resolve and reach the API server LB IP:

```bash
curl -k https://<lb-ip>:6443/healthz
# Should return: ok
```

If the node cannot reach the LB, check:
1. VNet peering or routing configuration
2. NSG rules on the control plane subnet (port 6443 must be open)
3. Load balancer health probe status: `az network lb show --resource-group $RESOURCE_GROUP --name lb-k8s-controlplane --query "probes"`

---

## 2. Cluster Autoscaler Not Scaling

### Symptom A: CA not scaling out despite pending pods

Pods remain in `Pending` state. CA logs show no scale-up events.

**Root Cause:** Tag misconfiguration or RBAC issues.

**Fix:**

1. Verify VMSS tags:
   ```bash
   az vmss show \
     --resource-group $RESOURCE_GROUP \
     --name vmss-k8s-workers \
     --query tags
   # Must include k8s.io/cluster-autoscaler/enabled=true
   # and k8s.io/cluster-autoscaler/<cluster-name>=owned
   ```

2. Check CA logs for "no node group found":
   ```bash
   kubectl -n kube-system logs -l app=cluster-autoscaler | grep -i "node group\|scale up\|no candidates"
   ```

3. Verify managed identity permissions:
   ```bash
   az role assignment list --assignee $MI_CLIENT_ID --output table
   ```

4. Test IMDS token acquisition from a control plane node:
   ```bash
   curl -s -H "Metadata: true" \
     "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/" \
     | jq .token_type
   # Should return: "Bearer"
   ```

### Symptom B: CA not scaling in

Nodes remain underutilized. CA logs show scale-down is blocked.

**Root Cause:** PodDisruptionBudgets, non-evictable pods, or cooldown period.

**Fix:**

```bash
# Check for non-evictable system pods
kubectl -n kube-system get pods -o wide | grep <node-name>

# Check PDBs
kubectl get pdb --all-namespaces

# Check CA scale-down log
kubectl -n kube-system logs -l app=cluster-autoscaler | grep -i "scale down\|unneeded\|blocked"

# If stuck in cooldown, check --scale-down-delay-after-add flag (default 10m)
kubectl -n kube-system get deployment cluster-autoscaler -o yaml | grep scale-down
```

---

## 3. ARM Throttling Errors at Scale

### Symptom

Cluster Autoscaler logs show `HTTP 429 Too Many Requests`. VMSS operations fail intermittently at large node counts.

### Root Cause

Azure Resource Manager enforces per-subscription rate limits. Large clusters making frequent VMSS reads/writes hit these limits, especially during cluster bootstrap or mass scale-out.

ARM limits (approximate):
- `GET` requests: 1,200 per hour per subscription per region
- `PUT`/`POST`/`DELETE` requests: 1,200 per hour per subscription per region

### Fix

1. **Increase CA cache TTL** to reduce ARM read frequency:
   ```bash
   # In CA deployment, add flags:
   --cloud-config-path=/etc/kubernetes/azure.json
   # In azure.json:
   "vmssCacheTTL": 120        # seconds (default 60)
   "vmssSizeRefreshPeriod": 60 # seconds (default 30)
   ```

2. **Use exponential backoff in automation scripts:**
   ```bash
   # The CA already implements backoff; do not call VMSS APIs from other tools simultaneously
   ```

3. **Distribute deployments across subscriptions** if you have >500 nodes and require frequent scaling.

4. **Monitor throttling via Azure Monitor:**
   ```bash
   az monitor metrics list \
     --resource "/subscriptions/$SUBSCRIPTION_ID" \
     --metric "Throttled Requests" \
     --namespace "Microsoft.Compute" \
     --interval PT1H
   ```

---

## 4. VMSS Auto-Repair Conflicting with Kubernetes Node Management

### Symptom

Nodes are unexpectedly deleted by VMSS while Kubernetes still thinks they are active. Workloads are interrupted without a graceful drain.

### Root Cause

VMSS Automatic Instance Repair monitors node health via the Application Health Extension. If the grace period is too short, VMSS deletes nodes that are temporarily unhealthy (e.g., during a kubelet restart or OS patch application) before Kubernetes can respond.

### Fix

1. **Increase grace period to at least 30 minutes:**
   ```bash
   az vmss update \
     --resource-group $RESOURCE_GROUP \
     --name $WORKER_VMSS_NAME \
     --automatic-repairs-grace-period 30
   ```

2. **Verify the health probe is checking kubelet, not a generic TCP port:**
   The Application Health Extension should probe port 10248 (`/healthz`), not port 22 or any other service.

3. **Temporarily disable auto-repair during maintenance:**
   ```bash
   az vmss update \
     --resource-group $RESOURCE_GROUP \
     --name $WORKER_VMSS_NAME \
     --enable-automatic-repairs false
   # Re-enable after maintenance
   az vmss update \
     --resource-group $RESOURCE_GROUP \
     --name $WORKER_VMSS_NAME \
     --enable-automatic-repairs true
   ```

---

## 5. Spot VM Nodes Disappearing Without Drain

### Symptom

Spot VM nodes are removed from the cluster abruptly. Running pods are not rescheduled gracefully. `kubectl get nodes` shows nodes going from `Ready` to `NotReady` to disappearing with no warning.

### Root Cause

The Spot Interruption Handler DaemonSet is not deployed, or the 30-second eviction window expires before the drain completes.

### Fix

1. **Deploy the Spot Interruption Handler** as documented in [Handle Spot VM Evictions](../how-to/handle-spot-evictions.md).

2. **Verify handler is running on Spot nodes:**
   ```bash
   kubectl -n kube-system get pods -l app=spot-interruption-handler -o wide
   ```

3. **Check handler logs for recent evictions:**
   ```bash
   kubectl -n kube-system logs -l app=spot-interruption-handler --tail=50
   ```

4. **Reduce pod `terminationGracePeriodSeconds`** on workloads on Spot nodes to ≤25 seconds.

5. **Test the handler** by triggering a simulated eviction:
   ```bash
   # Manually simulate — on the Spot node, check if handler responds to a test Preempt event
   # (No public API to simulate evictions; monitor in production)
   ```

---

## 6. GPU Nodes Not Showing `nvidia.com/gpu` Resource

### Symptom

GPU nodes are in `Ready` state but `kubectl describe node <gpu-node>` does not show `nvidia.com/gpu` in allocatable resources.

### Root Cause

GPU Operator components failed to install, or the NVIDIA driver failed to load.

### Fix

1. **Check GPU Operator pod status:**
   ```bash
   kubectl -n gpu-operator get pods
   # Look for: nvidia-driver-daemonset, nvidia-device-plugin-daemonset
   ```

2. **Check driver installation logs:**
   ```bash
   kubectl -n gpu-operator logs -l app=nvidia-driver-daemonset --tail=100
   ```

3. **Verify the driver loaded on the node:**
   ```bash
   # SSH into the GPU node
   lsmod | grep nvidia
   # If empty, driver is not loaded
   nvidia-smi
   ```

4. **Common cause: Secure Boot is enabled.** NVIDIA driver modules require signing for Secure Boot. Either disable Secure Boot or configure module signing:
   ```bash
   # Check Secure Boot status on node
   mokutil --sb-state
   ```

5. **Restart GPU Operator components:**
   ```bash
   kubectl -n gpu-operator rollout restart daemonset nvidia-driver-daemonset
   kubectl -n gpu-operator rollout restart daemonset nvidia-device-plugin-daemonset
   ```

---

## 7. kubeadm Join Fails After VMSS Scale-Out

### Symptom

New nodes added via VMSS scale-out fail to join with:
```
error execution phase preflight: couldn't validate the identity of the API Server: Get "https://<lb-ip>:6443/...": x509: certificate is valid for <existing-ip>, not <new-ip>
```

Or: `token ... is invalid, not found, or has expired`

### Root Cause A: Bootstrap token expired

`kubeadm` bootstrap tokens expire after 24 hours by default.

**Fix:**

Generate new tokens and update your cloud-init or custom script extension:

```bash
# Generate a new token
kubeadm token create --print-join-command

# Update the VMSS custom script extension with the new join command
az vmss extension set \
  --resource-group $RESOURCE_GROUP \
  --vmss-name $WORKER_VMSS_NAME \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings "{\"commandToExecute\": \"$(cat /path/to/updated-join-script.sh)\"}"
```

Consider using a short-lived token rotation mechanism or storing join credentials in Azure Key Vault with an automated rotation function.

### Root Cause B: Node IPs not in API server certificate SANs

**Fix:**

This should not occur if you're using the Load Balancer IP as the API server endpoint. If you used a node IP directly, re-run `kubeadm init` with the correct `--control-plane-endpoint`.

---

## 8. etcd Quorum Loss After Control Plane VM Replacement

### Symptom

API server is unresponsive. `kubectl` commands hang. One or more control plane VMs were replaced or reimaged. etcd logs show:
```
rafthttp: failed to send out heartbeat on time
etcdserver: failed to reach the quorum
```

### Root Cause

A 3-node etcd cluster can tolerate only 1 failed member. If 2 or more control plane VMs are replaced simultaneously (or if a replacement doesn't rejoin), quorum is lost and the cluster becomes read-only.

### Fix

**If one member is down** (quorum intact, cluster healthy):

```bash
# Remove the failed member
ETCD_POD=$(kubectl -n kube-system get pods -l component=etcd --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')

kubectl -n kube-system exec $ETCD_POD -- \
  etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Remove the failed member by ID
kubectl -n kube-system exec $ETCD_POD -- \
  etcdctl member remove <member-id> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# On the new/replacement VM, re-join as a new etcd member
sudo kubeadm join <lb-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>
```

**If quorum is lost** (cluster is down):

Restore from etcd snapshot. See [Upgrade Kubernetes — Rollback Procedure](../how-to/upgrade-kubernetes.md#rollback-procedure).

**Prevention:**

1. Never replace more than 1 control plane VM at a time
2. Verify the replacement has rejoined etcd before proceeding to the next
3. Take etcd snapshots before any control plane maintenance:
   ```bash
   # Schedule periodic etcd snapshots
   kubectl create -f - <<EOF
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: etcd-backup
     namespace: kube-system
   spec:
     schedule: "0 */6 * * *"
     jobTemplate:
       spec:
         template:
           spec:
             tolerations:
               - effect: NoSchedule
                 key: node-role.kubernetes.io/control-plane
             nodeSelector:
               node-role.kubernetes.io/control-plane: ""
             hostNetwork: true
             containers:
               - name: backup
                 image: registry.k8s.io/etcd:3.5.10-0
                 command:
                   - etcdctl
                   - snapshot
                   - save
                   - /backup/etcd-$(date +%Y%m%d-%H%M%S).db
                 env:
                   - name: ETCDCTL_API
                     value: "3"
                   - name: ETCDCTL_ENDPOINTS
                     value: "https://127.0.0.1:2379"
                   - name: ETCDCTL_CACERT
                     value: /etc/kubernetes/pki/etcd/ca.crt
                   - name: ETCDCTL_CERT
                     value: /etc/kubernetes/pki/etcd/server.crt
                   - name: ETCDCTL_KEY
                     value: /etc/kubernetes/pki/etcd/server.key
                 volumeMounts:
                   - name: etcd-certs
                     mountPath: /etc/kubernetes/pki/etcd
                     readOnly: true
                   - name: backup-storage
                     mountPath: /backup
             volumes:
               - name: etcd-certs
                 hostPath:
                   path: /etc/kubernetes/pki/etcd
               - name: backup-storage
                 persistentVolumeClaim:
                   claimName: etcd-backup-pvc
             restartPolicy: OnFailure
   EOF
   ```
