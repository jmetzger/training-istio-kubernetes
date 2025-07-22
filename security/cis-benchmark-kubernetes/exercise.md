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
```

```
# nodeName in template/spec: ergänzen wie folgt
spec:
  template:
    spec:
      nodeName: k8s-w1 
```

```
kubectl apply -f job.yaml
kubectl get pods -o wide 
# Durch Euren Pod ersetzen
# kubectl logs kube-bench-j76s9
# Oder: einfacher -> zeigt logs des 1. Pods des jobs
kubectl logs job/kube-bench
```

```
# Ausgabe
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
```

```
kubectl logs job/kube-bench > report.txt
```

## Beispiel für Fehlerbehebung auf Node 

### FAIL: 

```
[FAIL] 4.1.1 Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)
```

```
# Remediations
4.1.1 Run the below command (based on the file location on your system) on the each worker node.
For example, chmod 600 /lib/systemd/system/kubelet.service
```

### Fix: Walkthrough 

```
# ip - adresse ausfindig machen 
kubectl get nodes -o wide | grep cp 
```

## Reference:

  * https://hub.docker.com/r/aquasec/kube-bench

