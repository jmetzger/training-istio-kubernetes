# Linux Firewall for Cluster 

* `kubeadm` installation
* Calico (BGP mode)
* 1 control plane: `10.0.0.10`
* 3 workers: `10.0.0.11–13`
* All nodes in `10.0.0.0/24` subnet

---

## 🔐 Final Firewall Rule Summary

### ✅ Required Rules per Role

| Node Role     | Protocol | Port / Proto | From                 | Purpose                      |
| ------------- | -------- | ------------ | -------------------- | ---------------------------- |
| All nodes     | TCP      | 179          | other nodes          | Calico BGP peering           |
| Control plane | TCP      | 6443         | workers + admin      | Kubernetes API               |
| Control plane | TCP      | 2379-2380    | itself or HA members (control planes) | etcd                         |
| Control plane | TCP      | 10250        | workers              | Kubelet on CP                |
| Control plane | TCP      | 10257        | localhost            | kube-controller-manager      |
| Control plane | TCP      | 10259        | localhost            | kube-scheduler               |
| Worker nodes  | TCP      | 10250        | control plane        | Kubelet                      |
| Worker nodes  | TCP      | 30000–32767  | optional             | NodePort services            |
| All nodes     | TCP      | 22           | admin IP             | SSH (optional)               |

---


## ✅ Final Minimal Firewall Rules (kubeadm + Calico BGP, no Typha)

### 🔒 All Nodes (Control Plane + Workers)

```bash
# Allow Calico BGP peering
iptables -A INPUT -p tcp --dport 179 -s 10.0.0.0/24 -j ACCEPT
```
---

### 🧭 Control Plane Node (10.0.0.10)

```bash
# API Server
iptables -A INPUT -p tcp --dport 6443 -s 10.0.0.0/24 -j ACCEPT
# and additionally from client - infrastructure where you run kubectl 

# etcd
iptables -A INPUT -p tcp --dport 2379:2380 -s 10.0.0.0/24 -j ACCEPT

# Kubelet from workers
iptables -A INPUT -p tcp --dport 10250 -s 10.0.0.0/24 -j ACCEPT

# Scheduler & Controller Manager (localhost only)
iptables -A INPUT -p tcp --dport 10257 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 10259 -s 127.0.0.1 -j ACCEPT

# Admin SSH (optional)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.100 -j ACCEPT

# Allow established traffic
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Default deny
iptables -A INPUT -j DROP
```

---

### ⚙️ Worker Nodes (10.0.0.11–13)

```bash
# Kubelet access from control plane
iptables -A INPUT -p tcp --dport 10250 -s 10.0.0.10 -j ACCEPT

# NodePort services
iptables -A INPUT -p tcp --dport 30000:32767 -j ACCEPT

# Admin SSH (optional)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.100 -j ACCEPT

# Allow established traffic
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Default deny
iptables -A INPUT -j DROP
```

## Reference: Kubernetes Ports 

  * Achtung: Die Ports des CNI's (calico) kommen noch on-top
  * https://kubernetes.io/docs/reference/networking/ports-and-protocols/
