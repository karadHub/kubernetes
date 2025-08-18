# Google Kubernetes Engine (GKE)

This guide provides a beginner-friendly introduction to Google Kubernetes Engine (GKE), Google Cloud's managed Kubernetes service. We will cover the fundamental concepts, how to create and manage clusters, and how to deploy applications.

## Table of Contents

- [Prerequisites](#prerequisites)
- [GKE Overview](#gke-overview)
- [GKE Modes: Standard vs. Autopilot](#gke-modes-standard-vs-autopilot)
- [GKE Release Channels](#gke-release-channels)
- [Create a GKE Cluster (Standard Mode)](#create-a-gke-cluster-standard-mode)
- [GKE Cluster Walkthrough](#gke-cluster-walkthrough)
- [Create a GKE Cluster (Autopilot Mode)](#create-a-gke-cluster-autopilot-mode)
- [GKE Node Pools](#gke-node-pools)
- [Cluster Autoscaler vs. Node Auto-Provisioning](#cluster-autoscaler-vs-node-auto-provisioning)
- [Deploy a Sample "Hello World" App to GKE](#deploy-a-sample-hello-world-app-to-gke)
- [GKE Storage: Dynamic Provisioning](#gke-storage-dynamic-provisioning)
- [Deploying a Full-Stack App to GKE](#deploying-a-full-stack-app-to-gke)
- [Cleaning Up](#cleaning-up)
- [Official Documentation & Resources](#official-documentation--resources)

## Prerequisites

Before you begin, you need to have the Google Cloud SDK (`gcloud` CLI) installed and configured.

1.  **Install the Google Cloud SDK:** Follow the official installation instructions for your operating system.

2.  **Initialize the SDK:** Run the following command and follow the on-screen prompts to log in with your Google account and select a default project.

    ```bash
    gcloud init
    ```

3.  **Enable Required APIs:** You'll need to enable the Kubernetes Engine API for your project.

    ```bash
    gcloud services enable container.googleapis.com
    ```

## GKE Overview

Google Kubernetes Engine (GKE) is a managed environment for deploying, managing, and scaling your containerized applications using Google infrastructure. GKE abstracts away the complexity of managing the Kubernetes control plane (`kube-apiserver`, `etcd`, etc.), allowing you to focus on your applications.

Key benefits of GKE include:

- **Managed Control Plane:** Google manages, scales, and keeps the control plane up-to-date and healthy.
- **Scalability:** Easily scale your applications from a single instance to thousands, with features like cluster autoscaling.
- **Integration with Google Cloud:** Seamlessly integrates with other Google Cloud services like Monitoring, Logging, and IAM.
- **Security:** Provides a secure-by-default configuration and advanced security features.

> **Official Docs:** [GKE Technical Overview](https://cloud.google.com/kubernetes-engine/docs/concepts/kubernetes-engine-overview)

## GKE Modes: Standard vs. Autopilot

GKE offers two modes of operation, each catering to different needs for control and automation.

| Feature             | Standard Mode                                          | Autopilot Mode                                          |
| ------------------- | ------------------------------------------------------ | ------------------------------------------------------- |
| **Node Management** | You manage the underlying worker nodes and node pools. | GKE manages the nodes for you.                          |
| **Control**         | High degree of control over node configuration.        | Abstracted node management.                             |
| **Pricing Model**   | Pay for the worker nodes you provision.                | Pay for the CPU, memory, and storage your pods request. |
| **Responsibility**  | You are responsible for node security and upgrades.    | GKE handles node security and upgrades automatically.   |
| **Best For**        | Complex workloads requiring custom node configs.       | Most stateless and stateful web applications.           |

> **Official Docs:** [Comparing Autopilot and Standard](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview#comparison)

## GKE Release Channels

GKE automatically manages the version of your control plane and nodes. You can subscribe your cluster to a release channel to balance between feature availability and stability.

- **Rapid Channel:** Gets the newest features and versions first. Great for testing, but may have bugs.
- **Regular Channel:** A balance of new features and stability. Most new features undergo testing before being released here.
- **Stable Channel:** Receives the most mature and stable updates. Recommended for production clusters that are sensitive to change.

> **Official Docs:** [About Release Channels](https://cloud.google.com/kubernetes-engine/docs/concepts/release-channels)

## Create a GKE Cluster (Standard Mode)

After completing the prerequisites, you can create a GKE cluster.

This command creates a basic 3-node Standard cluster with autoscaling enabled.

```bash
# Set your project and default zone (optional, but recommended)
gcloud config set project YOUR_PROJECT_ID
gcloud config set compute/zone us-central1-c

# Create a GKE cluster in Standard mode
gcloud container clusters create "standard-cluster" \
    --num-nodes "1" \
    --enable-autoscaling --min-nodes "1" --max-nodes "3" \
    --release-channel "regular" \
    --zone "us-central1-c"

# After the cluster is created, get the credentials for kubectl
gcloud container clusters get-credentials standard-cluster --zone us-central1-c
```

## GKE Cluster Walkthrough

Once your cluster is running, you can inspect it using `kubectl`.

```bash
# Check the nodes in your cluster
kubectl get nodes
# OUTPUT:
# NAME                                                  STATUS   ROLES    AGE   VERSION
# gke-standard-cluster-default-pool-xxxx-yyyy   Ready    <none>   2m    v1.28.x-gke.x

# Check the pods running in the kube-system namespace (GKE managed components)
kubectl get pods -n kube-system
```

## Create a GKE Cluster (Autopilot Mode)

Creating an Autopilot cluster is even simpler because you don't need to specify node configurations.

```bash
# Create a GKE cluster in Autopilot mode
gcloud container clusters create-auto "autopilot-cluster" \
    --region "us-central1"

# Get the credentials for kubectl
gcloud container clusters get-credentials autopilot-cluster --region us-central1
```

## GKE Node Pools

In **Standard mode**, nodes are grouped into **Node Pools**. A node pool is a subset of machines within a cluster that all have the same configuration (machine type, disk size, etc.).

You might use multiple node pools to:

- Run a group of pods that need more CPU or memory on more powerful machines.
- Use preemptible VMs for batch jobs to save costs.
- Isolate different types of workloads from each other.

## Node Auto-Provisioning

Node Auto-Provisioning (NAP) is a feature for **Standard clusters** that goes beyond the cluster autoscaler. If you deploy a pod that cannot be scheduled on any existing node pool (e.g., it requires a specific machine type or architecture), NAP can automatically create a _new_ node pool that matches the pod's requirements.

This is different from the **Cluster Autoscaler**, which only adds or removes nodes within _existing_ node pools.

## Deploy a Sample "Hello World" App to GKE

Let's deploy a simple web server and expose it to the internet.

1.  **Create a file named `hello-app.yaml`:**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-web
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: hello
      template:
        metadata:
          labels:
            app: hello
        spec:
          containers:
            - name: hello-app
              image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
              ports:
                - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-web-service
    spec:
      type: LoadBalancer
      selector:
        app: hello
      ports:
        - protocol: TCP
          port: 80
          targetPort: 8080
    ```

2.  **Apply the manifest to your cluster:**

    ```bash
    kubectl apply -f hello-app.yaml
    ```

3.  **Check the status and get the external IP:**

    It might take a minute or two for the Load Balancer to get an external IP address.

    ```bash
    kubectl get service hello-web-service
    # Wait for EXTERNAL-IP to be populated
    # NAME                TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
    # hello-web-service   LoadBalancer   10.x.x.x      35.x.x.x        80:31234/TCP   60s
    ```

    You can now visit `http://<EXTERNAL-IP>` in your browser to see the "Hello, world!" message.

## Cluster Autoscaling

Cluster Autoscaling automatically resizes the number of nodes in a given node pool. When pods are pending because there aren't enough resources, the autoscaler adds nodes (up to the maximum you define). When nodes are underutilized, it removes them (down to the minimum).

This is enabled by default when using `gcloud container clusters create` with autoscaling flags like `--enable-autoscaling --min-nodes=1 --max-nodes=5`.

## GKE Storage: Dynamic Provisioning

GKE makes it easy to provision persistent storage for your stateful applications. GKE comes with pre-defined `StorageClass` objects that use Google Cloud Persistent Disks.

When you create a `PersistentVolumeClaim` (PVC), GKE's dynamic provisioner automatically creates a `PersistentVolume` (PV) and the underlying Persistent Disk for you.

**Example:** Create a PVC and a Pod that uses it.

1.  **Create a file named `storage-app.yaml`:**

    ```yaml
    # persistent-volume-claim (user requests dynamic provisioning)
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: my-gce-disk-claim
    spec:
      # The default storage class in GKE uses standard persistent disks.
      # You can also specify a storageClassName if you have multiple.
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
    ---
    # pod that uses the PVC
    apiVersion: v1
    kind: Pod
    metadata:
      name: app-with-storage
    spec:
      containers:
        - name: web-server
          image: nginx:stable
          volumeMounts:
            - mountPath: /var/www/html
              name: my-pd
      volumes:
        - name: my-pd
          persistentVolumeClaim:
            claimName: my-gce-disk-claim
    ```

2.  **Apply the manifest:**

    ```bash
    kubectl apply -f storage-app.yaml
    ```

    GKE will now provision a 10Gi Persistent Disk and attach it to the node where your `app-with-storage` pod is scheduled. The data you write to `/var/www/html` inside the container will persist even if the pod is restarted.

## Deploying a Full-Stack App to GKE

The following repository contains Kubernetes manifests for deploying a more complex, full-stack application.

```bash
git clone https://github.com/karadHub/K8S.git
```

After cloning the repository, you can explore the manifest files. Typically, you would apply them to your cluster using a command like `kubectl apply -f <directory-with-yaml-files>`.

## Summary

GKE simplifies Kubernetes management by handling the control plane and providing powerful automation features.

- Choose **Autopilot** for ease of use and to pay only for pod resources.
- Choose **Standard** when you need fine-grained control over node configurations.
- Use **Deployments** and **Services** to run and expose stateless applications.
- Use **PVCs** with GKE's default **StorageClass** for easy, dynamically provisioned persistent storage.
- Leverage **Cluster Autoscaling** and **Node Auto-Provisioning** to ensure your cluster has the right amount of resources for your workloads.

This updated `README.md` should serve as an excellent starting point for anyone learning GKE. Let me know if you have any other questions!

<!--
[PROMPT_SUGGESTION]Add a section explaining how to set up `gcloud` CLI for the first time.[/PROMPT_SUGGESTION]
[PROMPT_SUGGESTION]Create a new file explaining GKE networking concepts like Services, Ingress, and Gateway API.[/PROMPT_SUGGESTION]
