# Azure Kubernetes Service (AKS) — Beginner Friendly Guide

This document is a concise, beginner-friendly walkthrough of Azure Kubernetes Service (AKS).

It covers account setup, concepts, cluster creation (CLI + Portal), autoscaling, storage, deploying apps (including the sample app in this repo), and links to official docs for deeper reading.

## Checklist (what this guide covers)

- [AKS Overview](#1-aks-overview)
- [Create an Azure Trial Account](#2-create-an-azure-trial-account)
- [AKS: Standard vs serverless/virtual-node/autopilot options](#3-aks-standard-vs-serverless--virtual-nodes--autopilot)
- [AKS Upgrade channels](#4-aks-upgrade-channels)
- [AKS Node pools](#5-aks-node-pools)
- [NAP vs Cluster autoscaler](#6-nap-vs-cluster-autoscaler-conceptual)
- [Create AKS Cluster using CLI](#7-create-an-aks-cluster-using-the-cli-example)
- [AKS cluster presets](#8-aks-cluster-presets-templates-and-quickstarts)
- [AKS pricing overview](#9-aks-pricing-tier-overview)
- [Create a production cluster using the Azure Portal](#10-create-a-production-cluster-portal--recommended-settings)
- [Deploy a todo app to AKS (using the repo sample)](#11-deploy-a-todo-app-to-aks-quick-start-using-repository-sample)
- [Dynamic storage (StorageClass, PVCs, Azure Disk/Files)](#12-dynamic-storage-persistentvolumeclaims-and-storageclass)
- [Deploying a full-stack app to AKS](#13-deploying-a-full-stack-app-to-aks-practical-guidance)
- [Summary and official docs for reference](#14-summary-and-official-references)

---

## 1) AKS Overview

Azure Kubernetes Service (AKS) is a managed Kubernetes offering from Microsoft.

- AKS manages the Kubernetes control plane for you (free of charge) and lets you focus on deploying workloads.
- You still manage node pools (the VMs that run your pods) unless you opt for serverless or special modes.

Key benefits:

- Managed control plane (Azure handles upgrades, availability)
- Integration with Azure identity, monitoring, and networking
- Multiple node pools and VM choices

## 2) Create an Azure Trial Account

1. Visit https://azure.microsoft.com/free/ and sign up for a free trial.
2. You'll get a free credit and a limited-time subscription. Keep your subscription and billing info secure.
3. Install the Azure CLI locally: follow https://learn.microsoft.com/cli/azure/install-azure-cli

Quick CLI sign-in:

```bash
# Login interactively
az login

# If using multiple subscriptions, pick one
az account set --subscription "<SUBSCRIPTION_NAME_OR_ID>"
```

## 3) AKS Standard vs serverless / virtual nodes / Autopilot

- Standard AKS: You create and manage node pools (VMs). You choose VM sizes, counts, and scale settings. This is the most common and flexible mode for production.
- Virtual Nodes (serverless via Azure Container Instances): lets you burst pods into ACI without provisioning VMs. Good for spiky workloads but has limits.
- Autopilot / autoscale modes: Azure has features and add-ons to simplify management (preview features and marketplace offerings appear over time). For production, prefer Standard node pools for predictability.

If you see terms like "serverless" or "virtual nodes", they're about offloading compute to ACI for on-demand capacity.

## 4) AKS Upgrade channels

AKS provides upgrade channels that determine how frequently your cluster receives Kubernetes version updates (for example: rapid, stable, and patch channels). Pick:

- rapid — faster access to new versions (useful for testing)
- stable — balanced choice for most production clusters
- patch — only patch and security updates (conservative)

Recommendation: use `stable` for production clusters so upgrades are predictable.

## 5) AKS Node pools

- A node pool is a set of VMs with the same configuration (VM size, OS, labels/taints).
- There are system node pools (run system and control-plane-facing pods) and user node pools (run app pods).
- You can have multiple node pools: mix VM sizes, spot instances, GPU nodes, etc.
- Node pools can be scaled independently and support autoscaler.

Common operations:

- Add a node pool: `az aks nodepool add ...`
- Upgrade a node pool version independently
- Configure taints/labels for workload placement

## 6) NAP vs Cluster Autoscaler (conceptual)

- NAP (Node Auto-Provisioning) is a GKE concept where the control plane can _create new node pools automatically_ when needed. AKS does not have identical NAP behavior out-of-the-box.
- Cluster Autoscaler (AKS): scales existing node pools up/down by adding/removing VMs based on pending pods.

In AKS, common patterns to cover workload growth:

- Use Cluster Autoscaler on node pools to auto-scale existing pools.
- Combine with virtual nodes (ACI) for burst capacity.
- Use automation (scripts, AKS operator, or GitOps) to provision additional node pools if you need new VM sizes/types automatically.

## 7) Create an AKS cluster using the CLI (example)

This is a minimal, reproducible example. Adjust resource names, regions and sizes for your needs.

```bash
# Create resource group
az group create --name myAKS-rg --location eastus

# Create AKS cluster (basic, with managed identity and monitoring)
az aks create \
  --resource-group myAKS-rg \
  --name myAKSCluster \
  --node-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --kubernetes-version 1.26.9 \
  --node-vm-size Standard_DS2_v2

# Get kubectl credentials
az aks get-credentials --resource-group myAKS-rg --name myAKSCluster

# Verify nodes
kubectl get nodes
```

Advanced options you might add:

- `--network-plugin azure` (Azure CNI) for pod IPs in VNet
- `--enable-aad` to integrate Azure AD
- `--enable-managed-identity` for workload identity
- `--enable-cluster-autoscaler --min-count 1 --max-count 5` to enable autoscaling

To add a node pool:

```bash
az aks nodepool add --resource-group myAKS-rg --cluster-name myAKSCluster \
  --name np-gpu --node-count 1 --node-vm-size Standard_NC6
```

## 8) AKS Cluster Presets (templates and quickstarts)

- Use ARM templates, Bicep, or Terraform to codify cluster specs. These are the "presets" you'll reuse for consistent clusters.
- Azure Quickstart templates and the AKS sample gallery provide common starter configurations (multi-nodepool, AAD, private cluster, etc.).

Recommended approach: keep an IaC (infrastructure-as-code) template (Bicep/Terraform) for dev/staging/production presets.

## 9) AKS Pricing tier (overview)

- AKS control plane: free (you pay for the underlying VMs and resources).
- Costs you pay: VM instances (node pools), managed disks, load balancers (standard LB), public IP addresses, Azure Monitor logs/metrics, and any additional managed services (e.g., Azure Database, Container Registry storage).

Tips to control cost:

- Use spot instances for low-priority workloads.
- Choose appropriate VM sizes.
- Use scaling to shut down unnecessary nodes during off-hours.

## 10) Create a production cluster (Portal) — recommended settings

When using the Azure Portal, prefer these settings for production:

- Resource Group: dedicated RG per environment
- Region: choose an Azure region near users and with required SKUs
- Node pools: create at least 2 node pools (system + user) and use Availability Zones if supported
- VM size: choose size based on workload (CPU/memory/gpu)
- Network: Azure CNI for full VNet integration if you need advanced networking; otherwise Kubenet for simpler setups
- Security: enable RBAC, enable Azure AD integration, enable private cluster if you need isolated control plane
- Monitoring: enable Azure Monitor (Container insights)
- Upgrade channel: set to `stable` and enable automatic node image upgrades if you want automated OS image patching

Portal quick flow: Create resource > Kubernetes service > Basic settings (name/region) > Node pools > Authentication > Networking > Monitoring > Tags > Review + create.

## 11) Deploy a todo app to AKS (quick start using repository sample)

This repo includes a sample web app under `k8s-deployment-guide/sample-web-app`:

Files in the sample:

- `deployment.yaml` — Pod/deployment spec
- `service-*.yaml` — different Service types (ClusterIP, NodePort, LoadBalancer)

Deploy the sample app (after `kubectl` is configured for your AKS cluster):

```bash
# Use the manifest from the repo
kubectl apply -f k8s-deployment-guide/sample-web-app/deployment.yaml

# Expose with LoadBalancer (or apply one of the service files)
kubectl apply -f k8s-deployment-guide/sample-web-app/service-loadbalancer.yaml

# Check pods and service
kubectl get pods -n default
kubectl get svc -n default
```

If your cluster is private or behind an internal LB, use `kubectl port-forward` or configure an ingress controller.

## 12) Dynamic storage (PersistentVolumeClaims and StorageClass)

AKS supports dynamic provisioning via storage classes backed by Azure Disk or Azure Files.

Example PVC that requests an Azure managed disk (ReadWriteOnce):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: default
```

Common storage choices:

- Azure Disk (block storage, single pod mount for RWX limitations)
- Azure Files (SMB, multi-pod ReadWriteMany)

Check available StorageClasses:

```bash
kubectl get storageclass
```

If no suitable StorageClass exists you can create one that uses `managed-premium` or `managed-standard`.

## 13) Deploying a full-stack app to AKS (practical guidance)

For a full-stack app you typically need:

- Frontend deployment (React/Vue) served by an Ingress or CDN
- Backend API deployment (stateless microservices)
- Database: prefer a managed service (Azure Database for PostgreSQL/MySQL or Cosmos DB) for production instead of running DB inside AKS
- Storage: use PVCs for stateful components or Azure blob storage
- CI/CD: use GitHub Actions / Azure DevOps to build images and apply manifests or Helm charts
- Ingress & TLS: use NGINX ingress controller + cert-manager (Let's Encrypt)

Basic Helm flow example (install ingress + app):

```bash
# Add a chart repo and install (example)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace

# Deploy your app with a Helm chart or kubectl apply
helm install todo-app ./charts/todo-app
```

Considerations:

- Use separate namespaces per environment
- Use resource requests/limits
- Use liveness/readiness probes
- Use Horizontal Pod Autoscaler (HPA) for app-level scaling

## 14) Summary and official references

Summary:

- AKS is a managed Kubernetes service where you control node pools and Azure manages the control plane.
- Start with a small Standard cluster, enable monitoring, and use cluster autoscaler and node pools to grow.
- For production, use IaC (Bicep/Terraform), enable security features (AAD/RBAC/private cluster), and prefer managed databases for stateful services.

Official docs and further reading:

- AKS main docs: https://learn.microsoft.com/azure/aks/
- AKS quickstart (CLI): https://learn.microsoft.com/azure/aks/kubernetes-walkthrough-portal
- AKS storage concepts: https://learn.microsoft.com/azure/aks/concepts-storage
- AKS best practices: https://learn.microsoft.com/azure/aks/best-practices
- AKS autoscaler docs: https://learn.microsoft.com/azure/aks/cluster-autoscaler

---

If you want, I can also:

- Add a one-click Bicep/Terraform preset for a production cluster
- Create a Helm chart for the sample todo app in this repo
- Add step-by-step screenshots for portal instructions

Files used from this repo:

- `k8s-deployment-guide/sample-web-app/deployment.yaml` — example app deployment
- `k8s-deployment-guide/sample-web-app/service-*.yaml` — example service manifests

End of AKS beginner guide.
