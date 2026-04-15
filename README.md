# Self-Hosted Kubernetes on Azure VMSS

This guide covers everything you need to deploy, operate, and scale a self-managed Kubernetes cluster on Azure Virtual Machine Scale Sets (VMSS). It is written for platform engineers and infrastructure teams who need control beyond what Azure Kubernetes Service (AKS) provides.

---

## When to Use Self-Hosted Kubernetes vs. AKS

Use this guide when your requirements fall into one or more of these categories:

| Requirement | Self-Hosted K8s | AKS |
|---|---|---|
| Full control plane access (etcd, API server flags) | ✅ | ❌ |
| Custom Kubernetes versions not yet in AKS | ✅ | ❌ |
| GPUDirect RDMA / InfiniBand (NDv4, NDv5, H100) | ✅ | Partial |
| SR-IOV + Multus CNI for advanced networking | ✅ | Limited |
| Strict compliance (FedRAMP, DoD IL4/IL5) requiring full node ownership | ✅ | ❌ |
| Control plane HA across specific fault domains | ✅ | Managed by Azure |
| Custom admission controllers loaded at API server startup | ✅ | ❌ |
| Simplified operations, auto-upgrades, managed node pools | ❌ | ✅ |
| Minimal operational overhead | ❌ | ✅ |

**If none of the above apply, start with AKS.** AKS handles upgrades, node provisioning, and control plane management for you. Self-hosting is powerful but adds significant operational burden.

---

## Documentation Map

### Concepts
- [How VMSS Powers Self-Hosted Kubernetes](docs/concepts/vmss-for-kubernetes.md) — Fleet management, autoscaling, IMDS, Scheduled Events

### Quickstart
- [Deploy a Kubernetes Cluster on Azure VMSS Using kubeadm](docs/quickstart/deploy-kubeadm-vmss.md) — Get a working cluster in under an hour

### How-To Guides
- [Configure the Kubernetes Cluster Autoscaler with Azure VMSS](docs/how-to/configure-cluster-autoscaler.md)
- [Handle Spot VM Evictions Gracefully in Self-Hosted Kubernetes](docs/how-to/handle-spot-evictions.md)
- [Add GPU Worker Nodes to a Self-Hosted Kubernetes Cluster on VMSS](docs/how-to/gpu-nodes-vmss.md)
- [Upgrade Kubernetes on VMSS Without Downtime](docs/how-to/upgrade-kubernetes.md)
- [Configure Topology-Aware Scheduling for GPU Workloads](docs/how-to/topology-aware-scheduling.md)

### Reference
- [VMSS Configuration Reference for Kubernetes](docs/reference/vmss-configuration.md)

### Troubleshooting
- [Common Issues with Self-Hosted Kubernetes on VMSS](docs/troubleshooting/common-issues.md)

---

## Prerequisites Summary

Before you begin, ensure you have the following:

| Prerequisite | Minimum Version | Notes |
|---|---|---|
| Azure subscription | — | Contributor or Owner role on target resource group |
| Azure CLI (`az`) | 2.55.0+ | `az upgrade` to update |
| `kubectl` | 1.28+ | Must be within one minor version of your cluster |
| `kubeadm` | 1.28+ | Installed on all control plane and worker nodes |
| `containerd` | 1.7+ | Recommended container runtime |
| SSH access | — | Key pair for VM access during bootstrap |

Install the Azure CLI:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login
```

Install `kubectl`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

---

## Repository Structure

```
.
├── README.md
└── docs/
    ├── concepts/
    │   └── vmss-for-kubernetes.md
    ├── quickstart/
    │   └── deploy-kubeadm-vmss.md
    ├── how-to/
    │   ├── configure-cluster-autoscaler.md
    │   ├── handle-spot-evictions.md
    │   ├── gpu-nodes-vmss.md
    │   ├── upgrade-kubernetes.md
    │   └── topology-aware-scheduling.md
    ├── reference/
    │   └── vmss-configuration.md
    └── troubleshooting/
        └── common-issues.md
```

---

## Contributing

Pull requests are welcome. For major changes, open an issue first to discuss scope. All commands must be tested against real Azure infrastructure before merging.

## License

MIT
