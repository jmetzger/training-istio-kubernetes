# Exercise kube-bench (for control plane) - Scanning Cluster based on CIS Benchmark Kubernetes

## PreChecks / Cleanup 

```
# if you have a job kube-bench from the last exercise -> delete it
kubectl get jobs | grep kube-bench
# Only if it was there 
kubectl delete jobs kube-bench 
```

## Walkthrough

```
# Look for taints on master node
kubectl describe node k8s-cp
# easier
kubectl describe node k8s-cp | grep -i taints 
```

<img width="983" height="44" alt="image" src="https://github.com/user-attachments/assets/89e0dd91-a471-43bb-821c-a3771725ec13" />

```
# Make control-plane scheduable for now
kubectl taint nodes k8s-cp  node-role.kubernetes.io/control-plane:NoSchedule-
kubectl describe node k8s-cp | grep -i taints 
```

```
cd
mkdir -p manifests/kube-bench-cis-cp
cd manifests
cd kube-bench-cis-cp
```

```
wget https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

```
# nodeName in template/spec: ergänzen wie folgt
spec:
  template:
    spec:
      nodeName: k8s-cp
      containers:
        - command: ["kube-bench"]
 
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

## Beispiel für Fehlerbehebung auf Node (anonymous auth)

### Erklärung 

<img width="646" height="113" alt="image" src="https://github.com/user-attachments/assets/b9363593-d0a4-4b26-8d24-88bf3d4d2a6e" />

### FAIL: 

```
[FAIL] 1.2.15 Ensure that the --profiling argument is set to false (Automated)
```

```
# Remediations
1.2.15 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the control plane node and set the below parameter.
--profiling=false
```

### Fix: Walkthrough 

```
# ip - adresse ausfindig machen 
kubectl get nodes -o wide | grep k8s-cp
```

```
ssh 11trainingdo@<ip-des-control-plane-nodes>
```

```
sudo su -
```

```
cd /etc/kubernetes/manifests/
nano kube-apiserver.yaml
```

```
# changes from cis-scan 
- --profiling=false  # ergänzen
```

<img width="764" height="146" alt="image" src="https://github.com/user-attachments/assets/af1d9561-eba2-4e54-bfea-133475d4ed27" />

```
kubectl cluster-info
kubectl -n kube-system get pods
kubectl -n kube-system describe pods kube-apiserver
```

```
exit
exit
```


### After Fix: Walkhrough 

```
kubectl delete -f job.yaml
kubectl apply -f job.yaml
kubectl logs job/kube-bench > report-afterfix.txt
diff report.txt report-afterfix.txt
```

```
[PASS] 4.1.1 Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)
```

## Make it non-scheduable again 

```
kubectl taint nodes k8s-cp  node-role.kubernetes.io/control-plane:NoSchedule-
```


## Reference:

  * https://hub.docker.com/r/aquasec/kube-bench
  * https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

