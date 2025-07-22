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
```


### After Fix: Walkhrough 

```
kubectl delete -f job.yaml
kubectl apply -f job.yam
kubectl logs job/kube-bench > report-afterfix.txt 
```

```
[PASS] 4.1.1 Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)
```

## Reference:

  * https://hub.docker.com/r/aquasec/kube-bench

