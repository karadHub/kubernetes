# Kubernetes for Beginners

This repository contains notes and resources for learning Kubernetes.

## Table of Contents
- [Why Kubernetes?](#why-kubernetes)
- [What is a Pod?](#what-is-a-pod)
- [Kubernetes Architecture](#kubernetes-architecture)
- [Ways to Install Kubernetes](#ways-to-install-kubernetes)
- [Setup a Local Kubernetes Cluster on Linux](#setup-a-local-kubernetes-cluster-on-linux)

## Why Kubernetes?

In modern application development, containers (like those created with Docker) have become the standard way to package and run applications. They provide consistency across different environments. However, managing hundreds or thousands of containers manually across multiple machines is a significant challenge.

This is where **Container Orchestration** comes in. Kubernetes (often abbreviated as K8s) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

It solves key challenges by providing:
- **Service Discovery and Load Balancing:** Kubernetes can expose a container using a DNS name or their own IP address. If traffic to a container is high, Kubernetes can load balance and distribute the network traffic to stabilize the deployment.
- **Automated Rollouts and Rollbacks:** You can describe the desired state for your deployed containers using Kubernetes, and it will change the actual state to the desired state at a controlled rate. For example, you can automate Kubernetes to create new containers for your deployment, remove existing containers and adopt all their resources to the new container.
- **Self-healing:** Kubernetes restarts containers that fail, replaces containers, kills containers that don't respond to your user-defined health check, and doesn't advertise them to clients until they are ready to serve.
- **Storage Orchestration:** Kubernetes allows you to automatically mount a storage system of your choice, such as local storages, public cloud providers, and more.
- **Secret and Configuration Management:** Kubernetes lets you store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. You can deploy and update secrets and application configuration without rebuilding your container images and without exposing secrets in your stack configuration.

## What is a Pod?

A **Pod** is the smallest and simplest deployable unit in the Kubernetes object model. It represents a single instance of a running process in your cluster.

Key characteristics of a Pod:
- A Pod encapsulates one or more application containers (like Docker containers).
- Containers within the same Pod share the same network namespace, meaning they can communicate with each other using `localhost`.
- All containers in a Pod share the same storage volumes.
- Each Pod is assigned a unique IP address within the cluster.

While you can run a single container in a Pod (the most common use case), you can also run multiple tightly coupled containers that need to share resources. These are often called "sidecar" containers.

## Kubernetes Architecture

Kubernetes follows a client-server and a master-worker architecture. The main components are:

### Control Plane (formerly Master Node)
The Control Plane is the brain of the cluster. It makes global decisions about the cluster (e.g., scheduling) and detects and responds to cluster events.

- **`kube-apiserver`**: The frontend for the control plane. It exposes the Kubernetes API, which is how all other components and users interact with the cluster.
- **`etcd`**: A consistent and highly-available key-value store used as Kubernetes' backing store for all cluster data.
- **`kube-scheduler`**: Watches for newly created Pods and selects a node for them to run on based on resource requirements and other constraints.
- **`kube-controller-manager`**: Runs various controllers that handle cluster state. Examples include the Node Controller, Replication Controller, etc.

### Worker Nodes
Worker nodes are the machines (VMs, physical servers, etc.) where your applications (in containers) actually run.

- **`kubelet`**: An agent that runs on each worker node. It communicates with the `kube-apiserver` and ensures that containers described in PodSpecs are running and healthy.
- **`kube-proxy`**: A network proxy that runs on each node. It maintains network rules on nodes and enables network communication to your Pods from network sessions inside or outside of your cluster.
- **`Container Runtime`**: The software responsible for running containers. Kubernetes supports several container runtimes like Docker, `containerd`, and CRI-O.

## Ways to Install Kubernetes

There are several ways to get a Kubernetes cluster, depending on your needs:

1.  **Local Development Clusters:** Ideal for learning and local testing.
    - **Minikube:** Runs a single-node Kubernetes cluster inside a VM or Docker container on your local machine.
    - **Kind (Kubernetes in Docker):** Runs a multi-node cluster using Docker containers as "nodes".
    - **k3d / k3s:** A lightweight, certified Kubernetes distribution. `k3d` is a wrapper to run `k3s` in Docker.

2.  **Managed Kubernetes Services (Cloud Providers):** The easiest way to run production-grade clusters. The cloud provider manages the control plane for you.
    - **Google Kubernetes Engine (GKE)**
    - **Amazon Elastic Kubernetes Service (EKS)**
    - **Azure Kubernetes Service (AKS)**

3.  **Turnkey Solutions / On-Premises:** For deploying production clusters on your own infrastructure (bare metal or private cloud).
    - **`kubeadm`**: A tool to build a production-ready cluster from scratch.
    - **Kubespray**: Uses Ansible to deploy a production-ready cluster.
    - **Rancher**: A complete software stack for teams adopting containers.

## Setup a Local Kubernetes Cluster on Linux

Here's how to set up a local cluster using **Minikube** and **Docker** on a Linux system.

### 1. Install `kubectl`
`kubectl` is the command-line tool to interact with your Kubernetes cluster.

```bash
# Download the latest release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make the kubectl binary executable
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

### 2. Install Docker
Minikube can use Docker as the driver to run the Kubernetes components.

```bash
# Update package index and install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to the 'docker' group to run docker commands without sudo
sudo usermod -aG docker $USER && newgrp docker
```

### 3. Install Minikube

```bash
# Download the binary
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Make it executable and move to your path
chmod +x minikube
sudo mv minikube /usr/local/bin/
```

### 4. Start your Cluster

```bash
# Start minikube using the docker driver
minikube start --driver=docker

# Check the status
minikube status

# Verify cluster is running with kubectl
kubectl get nodes
```

You now have a single-node Kubernetes cluster running locally!