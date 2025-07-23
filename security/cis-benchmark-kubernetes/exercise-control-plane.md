# Exercise kube-bench (for control plane) - Scanning Cluster based on CIS Benchmark Kubernetes

## PreChecks / Cleanup 

```
# if you have a job kube-bench from the last exercise -> delete it
kubectl get jobs | grep kube-bench
# Only if it was there 
kubecl delete jobs kube-bench 
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
kubectl get nodes -o wide | grep k8s-w1
```

```
ssh 11trainingdo@<ip-des-w1-worker-nodes>
```

```
sudo su -
```

```
find /lib -name "kubelet.service"
find /usr/lib -name "kubelet.service"
```

```
chmod -R 600 /usr/lib/systemd/system/kubelet.service*

# Zur Überprüfung ob Rechteänderung auch für kubelet.service.d/kubeadm.conf
# gut ist -> weil wieder hochfährt 
systemctl restart kubelet
# ist er neu gestartet ?
systemctl status kubelet 
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


## Reference:

  * https://hub.docker.com/r/aquasec/kube-bench

