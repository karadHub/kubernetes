# Kubernetes for Beginners

Educational repo: core Kubernetes concepts, runnable sample manifests, and cloud provider comparisons for learners.

This repository contains notes and resources for learning Kubernetes.

## Table of Contents

- [Why Kubernetes?](#why-kubernetes)
- [What is a Pod?](#what-is-a-pod)
- [Kubernetes Architecture](#kubernetes-architecture)
- [Ways to Install Kubernetes](#ways-to-install-kubernetes)
- [Setup a Local Kubernetes Cluster on Linux](#setup-a-local-kubernetes-cluster-on-linux)
- [Kubernetes Storage](#kubernetes-storage)
- [Cloud Provider Comparison: GKE vs AKS vs EKS](comparisons/gke-aks-eks.md)
- [Quick Start](#quick-start)

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
## Kubernetes Storage

Understand how Kubernetes abstracts, provisions, attaches, secures, expands, and retires storage for Pods. This section goes beyond basics: volume categories, PV/PVC lifecycle, StorageClasses, CSI, snapshots, expansion, security, and best practices.

### 1. Volume categories

| Category | Examples (type) | Lifecycle | Typical use |
|----------|-----------------|-----------|-------------|
| Ephemeral (per Pod) | `emptyDir`, `configMap`, `secret`, `downwardAPI`, `projected`, `ephemeral` (CSI inline) | Deleted when Pod gone | Scratch, config injection, credentials |
| Node‑attached host path | `hostPath`, `local` (local PV) | Tied to specific node | Single-node fast IO, caches (avoid for portability) |
| Persistent (cluster) | `persistentVolumeClaim` (backed by cloud disks, NFS, filesystems, block, etc.) | Survives Pod restarts & rescheduling (until PV reclaim) | Databases, stateful apps |
| Special | `emptyDir{.medium=Memory}`, `tmpfs`, block mode PVs | Pod lifetime / PV lifetime | High-speed temp, raw block devices |

Notes:
* All containers in a Pod can mount declared volumes (optionally at different mount paths / readOnly settings).
* `subPath` lets you mount a subdirectory inside a volume (avoid dynamic path mutation at runtime—can break).

### 2. PV (PersistentVolume) & PVC (PersistentVolumeClaim)

* PV: Cluster-scoped object representing provisioned storage (static or dynamically created by a provisioner).
* PVC: Namespaced request for storage (size, access modes, optional StorageClass, volumeMode [Filesystem|Block], selectors).
* Binding: Control plane matches PVC -> PV (capacity, access modes, StorageClass, selectors). Status flows: Pending → Bound → (Released) → (Available / Failed).
* Volume expansion: If StorageClass `allowVolumeExpansion: true`, increasing PVC `spec.resources.requests.storage` (and applying) triggers expansion (filesystem grow may require Pod restart depending on driver).

### 3. Access modes (capability depends on backend)

* RWO (ReadWriteOnce): Single node read-write (cloud block disks: EBS gp3/gp2, GCE PD, Azure Disk).
* RWX (ReadWriteMany): Multiple nodes read-write (shared file: NFS, Azure Files, EFS, Filestore, CephFS).
* ROX (ReadOnlyMany): Multi-node read-only (rarely used intentionally—often fallback when sharing read-only media).
* RWOP (ReadWriteOncePod): Single Pod (even on same node) – stronger isolation (not all drivers support yet).

### 4. Provisioning models

Static provisioning:
* Admin pre-creates PV objects pointing to existing storage (NFS export, pre-provisioned disk, local path).

Dynamic provisioning (preferred):
* A `StorageClass` with a CSI driver provisioner automatically creates the backing volume when a PVC referencing that class appears.
* `volumeBindingMode: WaitForFirstConsumer` delays provisioning & node selection until a Pod using the PVC is scheduled (improves topology fit & AZ alignment).

### 5. StorageClass essentials

Fields: `provisioner`, `parameters`, `reclaimPolicy` (Delete|Retain), `allowVolumeExpansion`, `volumeBindingMode` (Immediate|WaitForFirstConsumer), `mountOptions`.

Example (AWS gp3 conceptually; adapt to your cloud):
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: fast-retain
provisioner: ebs.csi.aws.com          # cloud / CSI provisioner
parameters:
    type: gp3
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### 6. Reclaim policies

* Delete – Default for many dynamic classes; underlying storage deleted when PVC is removed and PV becomes Released.
* Retain – PV enters Released; data/manual cleanup required (safer for prod databases & forensics).
* Recycle – Deprecated (avoid).

### 7. CSI (Container Storage Interface)

Modern storage drivers are CSI-based (cloud disks, EFS/Azure Files/NFS, SAN vendors). Benefits: pluggable, features (snapshots, volume cloning, expansion, raw block, ephemeral inline).

Inspect installed drivers:
```bash
kubectl get csidrivers
```

### 8. Snapshots & cloning (if driver supports)

* VolumeSnapshot: Point-in-time capture of a PVC.
* Clone: Create a new PVC from an existing PVC (same StorageClass / driver).

Example snapshot objects (conceptual):
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
    name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
    name: db-snap-2025-08-21
spec:
    volumeSnapshotClassName: csi-snapclass
    source:
        persistentVolumeClaimName: db-data
```

### 9. Security & access

* Use least-privilege: prefer RWX only when multiple Pods truly need concurrent write.
* Set `fsGroup` (Pod `securityContext`) to ensure shared writable permission for non-root containers.
* Secrets & ConfigMaps are tmpfs; treat them as memory-backed (still visible in node memory & etcd unless encryption at rest enabled).
* Enable encryption-at-rest for etcd + provider-level disk encryption (KMS) for sensitive data.
* Avoid `hostPath` in multi-tenant clusters except for controlled system workloads.

### 10. Example: Static PV + PVC + Pod

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-local-1
spec:
    capacity:
        storage: 5Gi
    accessModes:
        - ReadWriteOnce
    storageClassName: ""           # Ensures it only binds to PVCs without a class
    persistentVolumeReclaimPolicy: Retain
    hostPath:
        path: /mnt/data/pv-local-1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-claim-1
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 5Gi
    storageClassName: ""            # Match the PV above
---
apiVersion: v1
kind: Pod
metadata:
    name: app-with-pv
spec:
    containers:
    - name: web
        image: nginx:stable
        volumeMounts:
        - name: app-storage
            mountPath: /usr/share/nginx/html
    volumes:
    - name: app-storage
        persistentVolumeClaim:
            claimName: pvc-claim-1
```

### 11. Example: Dynamic provisioning Deployment

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-dynamic-1
spec:
    storageClassName: standard
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
    name: web-dyn
spec:
    replicas: 1
    selector:
        matchLabels:
            app: web-dyn
    template:
        metadata:
            labels:
                app: web-dyn
        spec:
            containers:
            - name: nginx
                image: nginx:stable
                volumeMounts:
                - name: web-data
                    mountPath: /usr/share/nginx/html
            volumes:
            - name: web-data
                persistentVolumeClaim:
                    claimName: pvc-dynamic-1
```

### 12. Example: Volume expansion (after initial apply)

Edit PVC size (if `allowVolumeExpansion: true`):
```bash
kubectl patch pvc pvc-dynamic-1 -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
kubectl get pvc pvc-dynamic-1
```

### 13. Quick commands

```bash
# List storage classes & drivers
kubectl get storageclass
kubectl get csidrivers

# Describe PVC binding events
kubectl describe pvc pvc-dynamic-1

# Show PV lifecycle
kubectl get pv

# Snapshot list (if CRDs installed)
kubectl get volumesnapshotclasses 2>/dev/null || echo 'snapshot CRDs not installed'
```

### 14. Best practices & anti‑patterns

* Prefer dynamic provisioning; treat static PVs as niche (legacy NFS, special performance tuning, local PVs).
* Match access mode to actual need; unnecessary RWX increases blast radius & cost.
* Use `WaitForFirstConsumer` to avoid cross‑AZ scheduling failures in multi‑AZ clusters.
* Retain reclaim policy for critical state (databases) unless you have robust backup/snapshot automation.
* Implement regular backups (CSI snapshots + off-cluster copy) — snapshots alone are not off-site backups.
* Avoid embedding large data inside container images; mount persistent volumes or object storage.
* Monitor IOPS / throughput (cloud metrics) and set alerts; right-size before saturation.

Summary:
* Ephemeral: `emptyDir` (scratch) vs Persistent: PV/PVC (stateful data).
* StorageClasses + CSI enable on-demand, topology-aware provisioning, expansion, snapshots.
* Choose access modes, reclaim, and security context deliberately for resilience and least privilege.

---
- `emptyDir` is created when a Pod is assigned to a Node and exists as long as the Pod runs on that Node.
- It's useful for scratch space, caches, or for components that exchange files between containers in the same Pod.
- Data in `emptyDir` is lost if the Pod is evicted, deleted, or rescheduled to another Node.

3. Persistent Volumes and Persistent Volume Claims

- A PersistentVolume (PV) is a cluster-level resource representing storage provided by an administrator (static) or by the cluster through a provisioner (dynamic).
- A PersistentVolumeClaim (PVC) is a namespaced request for storage by a user. PVCs request size, access modes, and optionally a StorageClass.
- The control plane binds a PVC to a matching PV according to capacity, access modes, and StorageClass.

4. Access modes

- ReadWriteOnce (RWO): the volume can be mounted read-write by a single node.
- ReadOnlyMany (ROX): the volume can be mounted read-only by many nodes.
- ReadWriteMany (RWX): the volume can be mounted read-write by many nodes.
- Note: actual support for these modes depends on the underlying storage provider.

5. Static vs Dynamic Storage provisioning

- Static provisioning: An administrator creates PV objects that map to real storage (NFS exports, cloud disks, etc.). Users create PVCs that match available PVs.
- Dynamic provisioning: A StorageClass defines a provisioner and parameters. When a PVC requests a StorageClass, Kubernetes automatically creates a backing PV using the provisioner.

6. Reclaim policies

- Reclaim policies define what happens to the backing storage after a PV is released (the bound PVC is deleted):
  - Retain: Keep the data; manual cleanup and reuse required.
  - Delete: Delete the backing storage (commonly used with cloud disks).
  - Recycle: Deprecated—used to provide a basic scrub (not recommended).

7. StorageClass

- A `StorageClass` object provides parameters for dynamic provisioning (provisioner, parameters, reclaimPolicy, volumeBindingMode).
- Common fields: `provisioner`, `parameters`, `reclaimPolicy` (Delete/Retain), and `volumeBindingMode` (Immediate/WaitForFirstConsumer).
- Use `kubectl get storageclass` to see available classes in the cluster.

8. Examples — provisioning a Pod with persistent storage

Below are two compact examples:

- Static provisioning (admin creates a PV, a user creates a PVC and a Pod uses the PVC):

```yaml
# --- persistent-volume (admin)
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-local-1
spec:
    capacity:
        storage: 5Gi
    accessModes:
        - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    hostPath:
        path: /mnt/data/pv-local-1

# --- persistent-volume-claim (user)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-claim-1
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 5Gi

# --- pod that uses the PVC
apiVersion: v1
kind: Pod
metadata:
    name: app-with-pv
spec:
    containers:
    - name: web
        image: nginx:stable
        volumeMounts:
        - mountPath: /usr/share/nginx/html
            name: app-storage
    volumes:
    - name: app-storage
        persistentVolumeClaim:
            claimName: pvc-claim-1
```

- Dynamic provisioning (cluster has a StorageClass named `standard`):

```yaml
# --- persistent-volume-claim (user requests dynamic provisioning)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-dynamic-1
spec:
    storageClassName: standard
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 10Gi

# --- deployment that uses the dynamically provisioned PVC
apiVersion: apps/v1
kind: Deployment
metadata:
    name: web-dyn
spec:
    replicas: 1
    selector:
        matchLabels:
            app: web-dyn
    template:
        metadata:
            labels:
                app: web-dyn
        spec:
            containers:
            - name: nginx
                image: nginx:stable
                volumeMounts:
                - name: web-data
                    mountPath: /usr/share/nginx/html
            volumes:
            - name: web-data
                persistentVolumeClaim:
                    claimName: pvc-dynamic-1
```

Summary

- Use `emptyDir` for ephemeral, per-Pod scratch space. Use PV + PVC when you need data to persist beyond Pod lifetimes.
- Prefer dynamic provisioning with a `StorageClass` in cloud or well-administered clusters for simplicity and automation.
- Choose access modes and reclaim policies based on your application needs and your storage backend capabilities.

Quick commands

```bash
# See storage classes
kubectl get storageclass

# Create PVC and check binding
kubectl apply -f pvc-dynamic-1.yaml
kubectl get pvc pvc-dynamic-1
kubectl describe pvc pvc-dynamic-1
```

---

## Cloud Provider Comparison

Moved to a dedicated document: [GKE vs AKS vs EKS](comparisons/gke-aks-eks.md) to keep this introduction concise.
