# Kubernetes Deployment and Service Guide

This guide explains how to deploy a simple web application to Kubernetes using a Deployment and expose it using different types of Services.

## What is a Deployment?

A **Deployment** is a Kubernetes object that manages a set of identical Pods. It provides declarative updates to Pods along with many other useful features. In essence, you describe a *desired state* in a Deployment object, and the Deployment Controller changes the actual state to the desired state at a controlled rate.

Key features of a Deployment:
- **Manages ReplicaSets**: A Deployment manages a ReplicaSet, which in turn ensures that a specified number of Pod "replicas" are running at any given time.
- **Rolling Updates**: You can update the application without downtime. The Deployment ensures that only a certain number of Pods are down while they are being updated, and that new Pods are running before the old ones are terminated.
- **Rollbacks**: If an update has issues, you can easily revert to a previous version of your application.

### Sample Deployment (`deployment.yaml`)

The `deployment.yaml` file in this directory defines a Deployment that runs three replicas of a simple Python web application.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app-deployment
  labels:
    app: python-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: python-web-app
  template:
    metadata:
      labels:
        app: python-web-app
    spec:
      containers:
      - name: web-server
        image: karadhub/python-web-app
        ports:
        - containerPort: 8000
```

## What is a Service?

A **Service** is a Kubernetes object that defines a logical set of Pods and a policy by which to access them. Since Pods are ephemeral (they can be created and destroyed), a Service provides a stable endpoint (a single IP address and DNS name) to access the application running in those Pods.

Services use `selectors` to find the Pods they should route traffic to. There are three main types of Services:

### 1. ClusterIP Service

This is the default Service type. It exposes the Service on an internal IP address within the cluster. This makes the Service reachable only from *within* the cluster. It's ideal for internal communication between different parts of your application (e.g., a web frontend talking to a backend API).

See `service-clusterip.yaml` for an example.

### 2. NodePort Service

This Service type exposes the application on a static port on each of the cluster's Worker Nodes. A `ClusterIP` Service is automatically created, and the `NodePort` Service will route traffic to it. You can access the application from outside the cluster by requesting `<NodeIP>:<NodePort>`. This is useful for development or when you don't have a cloud load balancer.

See `service-nodeport.yaml` for an example.

### 3. LoadBalancer Service

This Service type exposes the application externally using a cloud provider's load balancer. It's the standard way to expose an application to the internet in a production environment. When you create a `LoadBalancer` Service, a `NodePort` and `ClusterIP` Service are automatically created, and the external load balancer will route traffic to them.

This type only works if you are running your cluster on a cloud provider (like GKE, EKS, or AKS) that can provision an external load balancer. On local clusters like Minikube, you can use the `minikube tunnel` command to make it work.

See `service-loadbalancer.yaml` for an example.


## How to Deploy the Application

```makdown
Folder Structure/
├── deployment.yaml
├── service-clusterip.yaml
├── service-nodeport.yaml
├── service-loadbalancer.yaml
```

Follow these steps to deploy the application and expose it.

### 1. Apply the Deployment

This will create the Pods running the application.

```bash
kubectl apply -f deployment.yaml

# Check that the deployment is successful and pods are running
kubectl get deployment
kubectl get pods
```

### 2. Expose the Deployment with a Service

Choose one of the following service types to expose your deployment.

#### Using NodePort

This is the easiest way to test on a local cluster like Minikube.

```bash
# Apply the NodePort service
kubectl apply -f service-nodeport.yaml

# Find the URL to access your service
minikube service python-app-nodeport-svc --url
```

#### Using LoadBalancer (with Minikube)

If you want to test the LoadBalancer service locally, you need to run `minikube tunnel` in a separate terminal.

```bash
# In a new terminal, start the tunnel
minikube tunnel

# In your original terminal, apply the service
kubectl apply -f service-loadbalancer.yaml

# Get the external IP of the service (it may take a minute)
kubectl get service python-app-lb-svc
```

