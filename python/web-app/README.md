# Sample Web App (Python)

Simple illustrative deployment + services showing ClusterIP, NodePort (and optional LoadBalancer) with kustomize grouping.

## Files

- `deployment.yaml` – 3 replica web Deployment with probes
- `service-clusterip.yaml` – internal service
- `service-nodeport.yaml` – external via node port (dev/test)
- `service-loadbalancer.yaml` – optional cloud LB
- `kustomization.yaml` – bundle resources

## Apply

```bash
kubectl apply -k .
```

## Remove

```bash
kubectl delete -k .
```

## Access

NodePort:

```bash
kubectl get svc python-app-nodeport-svc
```

LoadBalancer (if enabled):

```bash
kubectl get svc python-app-lb-svc -w
```
