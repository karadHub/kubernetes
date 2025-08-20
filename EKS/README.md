# EKS (Amazon Elastic Kubernetes Service) Beginner Guide

This guide walks a beginner through Amazon EKS concepts and hands‑on steps: account setup, cluster creation (CLI & Console, Auto Mode vs standard), node groups, versioning, autoscaling, storage, and deploying sample (todo + full‑stack) apps.

## Table of Contents

1. [EKS Overview](#1-eks-overview)
2. [Create an AWS Free Tier / Trial Account](#2-create-an-aws-free-tier--trial-account)
3. [EKS Deployment Modes (Standard vs Auto Mode)](#3-eks-deployment-modes-standard-vs-auto-mode)
4. [EKS Auto Mode Explained](#4-eks-auto-mode-explained)
5. [Create an EKS Cluster via CLI (eksctl)](#5-create-an-eks-cluster-via-cli-eksctl)
6. [Create / Manage Node Groups](#6-create--manage-node-groups)
7. [EKS Cluster Walkthrough (Key Components)](#7-eks-cluster-walkthrough-key-components)
8. [Kubernetes Version Support in EKS](#8-kubernetes-version-support-in-eks)
9. [Create an EKS Cluster via Console (Standard Mode)](#9-create-an-eks-cluster-via-console-standard-mode)
10. [Create an EKS Cluster via Console (Auto Mode)](#10-create-an-eks-cluster-via-console-auto-mode)
11. [Deploy a Todo App to EKS](#11-deploy-a-todo-app-to-eks)
12. [Cluster Autoscaling in EKS](#12-cluster-autoscaling-in-eks)
13. [Dynamic Storage in EKS (EBS & EFS)](#13-dynamic-storage-in-eks-ebs--efs)
14. [Deploying a Full‑Stack App to EKS](#14-deploying-a-fullstack-app-to-eks)
15. [Cleanup (Avoiding Unwanted Costs)](#15-cleanup-avoiding-unwanted-costs)
16. [Summary](#16-summary)
17. [Official References](#17-official-references)

## 1. EKS Overview

Amazon EKS is a managed control plane for Kubernetes. AWS runs & scales the control plane (API server, etcd) across Availability Zones for high availability; you manage worker compute (unless using Auto Mode). You get:

- Automated control plane upgrades & patching.
- Native integration (IAM, VPC, ELB, CloudWatch, ECR, KMS, GuardDuty, WAF, etc.).
- Choice of EC2 nodes, Fargate (serverless pods), and EKS Auto Mode (on‑demand managed data plane).

Core concepts:

- Cluster: Control plane + cluster networking (VPC subnets) + data plane (nodes).
- Node group: Set of EC2 instances (or managed data plane) with identical config.
- Pod: Smallest deployable unit (containers + specs).
- Service: Stable networking abstraction (ClusterIP, NodePort, LoadBalancer).
- Ingress: Layer 7 routing (optional, via controller like ALB Ingress Controller).

## 2. Create an AWS Free Tier / Trial Account

Steps:

1. Go to https://aws.amazon.com/free/ and click Create an AWS Account.
2. Use a unique email and secure password (enable MFA early).
3. Provide billing info (a small temporary authorization may appear).
4. Choose Support Basic (free) unless you need premium.
5. After activation: set up IAM:
   - Create an IAM user with AdministratorAccess (avoid using root).
   - Enable MFA on root & admin.
   - Create an access key for CLI only if needed (prefer AWS SSO for enterprises).
6. Region: pick one near you with EKS support (e.g. us-east-1, us-west-2, eu-west-1).

Install prerequisites locally:

- AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- kubectl: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
- eksctl: https://eksctl.io/introduction/installation/
- (Optional) Helm: https://helm.sh/docs/intro/install/

Configure credentials:

```bash
aws configure                # Provide Access Key, Secret, region, output (json)
aws sts get-caller-identity  # Verify identity
```

## 3. EKS Deployment Modes (Standard vs Auto Mode)

EKS now offers two broad approaches:

- Standard Mode: You provision networking (VPC/subnets) and data plane (managed node groups, self-managed nodes, Fargate profiles). Most control and flexibility.
- Auto Mode: EKS automatically manages cluster compute, networking constructs, and scaling behind the scenes. Reduced operational overhead; fewer tunables.

Comparison (high level):

- Provisioning Speed: Auto faster (minutes) vs Standard (10–15 min typical).
- Cost Transparency: Standard explicit EC2/Fargate billing; Auto abstracts some dimensions.
- Custom AMI / DaemonSets: Standard full control; Auto restricted (some DaemonSets may be disallowed).
- Advanced Networking (custom CNI tweaks): Standard; limited in Auto.
- Suitable for: Auto (rapid dev/test, small teams). Standard (prod w/ custom security, perf, cost tuning).

## 4. EKS Auto Mode Explained

Auto Mode simplifies:

- Data plane selection & right‑sizing.
- Autoscaling and placement.
- Networking (cluster IP space & CNI config).
  You still interact with Kubernetes API normally (kubectl, Helm). Some low-level tweaks (custom kernel params, privileged host access) are not available.

## 5. Create an EKS Cluster via CLI (eksctl)

`eksctl` is the quickest CLI to stand up a standard-mode cluster.

Minimal config file (`cluster.yaml`):

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
	name: demo-eks
	region: us-east-1
	version: "1.30"
vpc:
	nat:
		gateway: Single
managedNodeGroups:
	- name: ng-general
		instanceType: t3.medium
		desiredCapacity: 2
		minSize: 2
		maxSize: 5
		volumeSize: 20
		ssh:
			allow: true
			publicKeyName: my-keypair
cloudWatch:
	clusterLogging:
		enableTypes: ["api", "audit", "authenticator"]
```

Create:

```bash
eksctl create cluster -f cluster.yaml
```

Verify:

```bash
kubectl get nodes
kubectl get pods -A
```

Update kubeconfig manually (if needed):

```bash
aws eks update-kubeconfig --region us-east-1 --name demo-eks
```

## 6. Create / Manage Node Groups

Add another managed node group later:

```bash
eksctl create nodegroup \
	--cluster demo-eks \
	--name ng-spot \
	--node-type t3.large \
	--nodes 2 --nodes-min 1 --nodes-max 4 \
	--spot
```

List:

```bash
eksctl get nodegroup --cluster demo-eks
```

Delete:

```bash
eksctl delete nodegroup --cluster demo-eks --name ng-spot
```

## 7. EKS Cluster Walkthrough (Key Components)

Inspect namespaces:

```bash
kubectl get ns
```

System components (in `kube-system`):

- `aws-node` (Amazon VPC CNI) – Pod networking.
- `coredns` – DNS.
- `kube-proxy` – Service networking (iptables/ipvs).

Describe cluster details:

```bash
aws eks describe-cluster --name demo-eks --region us-east-1 --query 'cluster.{name:name,version:version,status:status,endpoint:endpoint}'
```

## 8. Kubernetes Version Support in EKS

EKS typically supports at least 4 active minor versions. When a version approaches end-of-life:

1. AWS announces deprecation timeline.
2. You should upgrade (control plane then node groups).
   Check available versions:

```bash
aws eks describe-cluster --name demo-eks --query 'cluster.version'
aws eks describe-addon-versions --addon-name vpc-cni | head
```

Upgrade (example 1.29 -> 1.30):

```bash
eksctl upgrade cluster --name demo-eks --approve
eksctl upgrade nodegroup --cluster demo-eks --name ng-general --kubernetes-version 1.30
```

## 9. Create an EKS Cluster via Console (Standard Mode)

AWS Console > EKS > Add cluster > Create.
Steps:

1. Name + version.
2. Specify cluster IAM service role (create if absent: AmazonEKSClusterPolicy, VPC resource permissions).
3. Networking: select or create VPC with 2+ public & private subnets across AZs.
4. Logging (optional) – enable API/Audit for debugging.
5. Create, wait Active.
6. Add Node group: specify IAM node role (AmazonEKSWorkerNodePolicy, AmazonEC2ContainerRegistryReadOnly, AmazonEKS_CNI_Policy), instance types, scaling, update strategy.
7. Update kubeconfig: copy command shown in console.

## 10. Create an EKS Cluster via Console (Auto Mode)

Console > EKS > Add cluster > Auto mode:

1. Provide name & version.
2. Select region; networking auto-generated.
3. Optionally configure access entries (IAM users / roles to map).
4. Review & create.
   Data plane is provisioned automatically; you do not manage node groups manually.

## 11. Deploy a Todo App to EKS

Prereq: Build/push an image to ECR or use a public image (example uses `nginx` placeholder). Minimal manifests:
`todo-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: todo-web
spec:
	replicas: 2
	selector:
		matchLabels:
			app: todo-web
	template:
		metadata:
			labels:
				app: todo-web
		spec:
			containers:
				- name: web
					image: public.ecr.aws/nginx/nginx:latest
					ports:
						- containerPort: 80
					env:
						- name: APP_MODE
							value: demo
```

`todo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
	name: todo-web-svc
spec:
	type: LoadBalancer
	selector:
		app: todo-web
	ports:
		- port: 80
			targetPort: 80
```

Apply:

```bash
kubectl apply -f todo-deployment.yaml -f todo-service.yaml
kubectl get svc todo-web-svc -w
```

Access via EXTERNAL-IP.

## 12. Cluster Autoscaling in EKS

Two layers:

- Horizontal Pod Autoscaler (HPA) – adjusts replicas.
- Cluster Autoscaler (CA) – adjusts node group size.

Install metrics server (if not present):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Example HPA:

```bash
kubectl autoscale deployment todo-web --cpu-percent=50 --min=2 --max=6
```

Cluster Autoscaler (managed node groups):

1. Tag the node group ASG(s) (eksctl does this automatically) with scaling tags.
2. Deploy CA via Helm:

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
	-n kube-system \
	--set autoDiscovery.clusterName=demo-eks \
	--set awsRegion=us-east-1 \
	--set rbac.serviceAccount.create=true \
	--set extraArgs.balance-similar-node-groups=true \
	--set extraArgs.skip-nodes-with-system-pods=false
```

Annotate deployment for log-friendly behavior (optional) and set correct image tag matching Kubernetes minor version.

## 13. Dynamic Storage in EKS (EBS & EFS)

Install AWS EBS CSI driver (addon):

```bash
eksctl utils install-vpc-controllers --cluster demo-eks # (legacy) or use addon below
aws eks create-addon --cluster-name demo-eks --addon-name aws-ebs-csi-driver
```

StorageClass (gp3 example) usually auto-created; list:

```bash
kubectl get storageclass
```

Sample PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: data-pvc
spec:
	accessModes: ["ReadWriteOnce"]
	resources:
		requests:
			storage: 5Gi
	storageClassName: gp3
```

Mount in Pod via `volumeMounts` + `volumes` referencing `claimName`.

For shared storage, deploy EFS CSI driver (requires EFS file system in same VPC) then use `ReadWriteMany` PVC with the EFS storage class.

## 14. Deploying a Full‑Stack App to EKS

Typical pattern:

1. Build & push images (frontend, api, worker) to ECR repo(s).
2. Create Kubernetes manifests / Helm chart: Deployments, Services, ConfigMaps/Secrets, Ingress.
3. Use separate namespaces (e.g., `prod`, `staging`).
4. Configure Ingress Controller (AWS Load Balancer Controller) for path/host routing.
5. Add CI/CD (GitHub Actions) to build & deploy (kubectl / Helm).

Minimal multi-tier example snippet (conceptual):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: api
spec:
	replicas: 2
	selector:
		matchLabels:
			app: api
	template:
		metadata:
			labels:
				app: api
		spec:
			containers:
				- name: api
					image: <account>.dkr.ecr.us-east-1.amazonaws.com/api:1.0.0
					envFrom:
						- secretRef:
								name: api-secrets
					ports:
						- containerPort: 8080
```

Add Service + Ingress for frontend & api. Store secrets in AWS Secrets Manager or SSM Parameter Store (optionally integrate via external secrets operator).

## 15. Cleanup (Avoiding Unwanted Costs)

Delete sample workloads first:

```bash
kubectl delete -f todo-service.yaml -f todo-deployment.yaml
```

Delete cluster (eksctl):

```bash
eksctl delete cluster --name demo-eks
```

Verify no leftover: ELBs (EC2 > Load Balancers), EBS volumes, ECR images, CloudWatch logs, EFS, NAT gateways (costly), and any Route 53 records.

## 16. Summary

You learned: what EKS is, account setup, creating clusters (CLI/Console, standard vs Auto Mode), node groups, version management, app deployment, autoscaling, and dynamic storage. Standard Mode offers granular control; Auto Mode accelerates experimentation. Next: layer on observability (Prometheus/Grafana), security (IAM Roles for Service Accounts), GitOps (Argo CD or Flux), and cost optimization (Spot, rightsizing, cluster autoscaler).

## 17. Official References

- EKS Main Docs: https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
- eksctl Docs: https://eksctl.io/
- Pricing: https://aws.amazon.com/eks/pricing/
- Kubernetes Docs: https://kubernetes.io/docs/home/
- AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/
- Cluster Autoscaler: https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler
- EBS CSI Driver: https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
- EFS CSI Driver: https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
- Version calendar: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html

## Notes

- Always tag resources for tracking (e.g., `env=dev`, `app=demo`).
- Keep IAM least privilege (scoped roles for CI/CD, read-only roles for observers).
- Enable control plane logging only as needed to control CloudWatch costs.

Happy shipping on EKS.
