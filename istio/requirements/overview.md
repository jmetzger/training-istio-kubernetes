# Systemanforderungen & Kubernetes-Cluster-Vorbereitung

## Hardware-Anforderungen

### Master Node (Control Plane)
- **Minimum:** 2 CPU, 2 GB RAM, 20 GB Disk
- **Empfohlen:** 4 CPU, 8 GB RAM, 50 GB Disk
- **Schulung:** 2 CPU, 4 GB RAM, 30 GB Disk

### Worker Node
- **Minimum:** 2 CPU, 2 GB RAM, 20 GB Disk
- **Empfohlen:** 4 CPU, 16 GB RAM, 100 GB Disk
- **Schulung:** 2 CPU, 4 GB RAM, 30 GB Disk

## Software-Voraussetzungen

### Betriebssystem
- Ubuntu 20.04/22.04 LTS
- Debian 11/12
- RHEL/Rocky/Alma Linux 8/9
- 64-bit Architektur erforderlich

### Container Runtime
- containerd (empfohlen)
- CRI-O
- Docker Engine (über cri-dockerd)

### System-Tools
```bash
curl, wget, apt-transport-https, ca-certificates
```

## Netzwerk-Anforderungen

### Ports Control Plane
- 6443: Kubernetes API Server
- 2379-2380: etcd
- 10250: Kubelet API
- 10259: kube-scheduler
- 10257: kube-controller-manager

### Ports Worker Nodes
- 10250: Kubelet API
- 30000-32767: NodePort Services

## Cluster-Vorbereitung

### System-Updates
```bash
apt update && apt upgrade -y
# oder
dnf update -y
```

### Swap deaktivieren
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

### Kernel-Module laden
```bash
cat <<EOL | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOL

modprobe overlay
modprobe br_netfilter
```

### Sysctl-Parameter
```bash
cat <<EOL | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOL

sysctl --system
```

### Firewall
- UFW/firewalld deaktivieren (Schulung)
- Oder: Benötigte Ports freischalten



lsmod | grep br_netfilter
```
