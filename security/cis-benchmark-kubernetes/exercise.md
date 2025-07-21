# Exercise kube-bench - Scanning Cluster based on CIS Benchmark Kubernetes

## Walkthrough

```
cd
mkdir -p manifests/kube-bench-cis 
cd manifests
cd kube-bench-cis
```

```
wget https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl apply -f job.yaml
kubectl get pods
# Durch Euren Pod ersetzen
kubectl logs kube-bench-j76s9
```

```
# Ausgabe
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
```

## Reference:

  * https://hub.docker.com/r/aquasec/kube-bench

