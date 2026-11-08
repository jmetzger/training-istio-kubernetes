# Install istio with istioctl (ambient) 

Getestet mit Istio 1.31.4, Kubernetes v1.37 und Calico (Iptables-Dataplane).

## Step 1: Gateway API CRDs

Ab Istio 1.30 werden die Gateway API CRDs in Version v1.5.x benötigt.
Ohne sie werden z.B. TLSRoute und ReferenceGrant von istiod nicht gesehen.

```
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

## Step 2: Installation von istio (Profil ambient)

```
istioctl version --remote=false
istioctl install --set profile=ambient --skip-confirmation
```

## Step 3: Überprüfen

```
kubectl -n istio-system get pods
istioctl version
```

Erwartet: istiod (1x), istio-cni-node (1x pro Node) und ztunnel (1x pro Node) laufen,
Client, Control Plane und Data Plane melden dieselbe Version.

## Hinweis zu Calico

Bei der Installation kann diese Meldung erscheinen:

```
detected Calico CNI with 'bpfConnectTimeLoadBalancing=TCP'; this must be set to 'bpfConnectTimeLoadBalancing=Disabled'
```

  * Sie betrifft nur Calico im **eBPF-Modus** (`bpfEnabled: true`).
  * Mit dem Iptables-Dataplane (so sind die Trainings-Cluster aufgesetzt) ist die Einstellung wirkungslos,
    die Meldung kann ignoriert werden. Getestet: Bookinfo im Ambient-Namespace, mTLS und STRICT funktionieren.

```
# Prüfen, welcher Dataplane läuft
kubectl get felixconfiguration default -o jsonpath='{.spec.bpfEnabled}{"\n"}'
```

Nur wenn das `true` ausgibt:

```
kubectl patch felixconfiguration default --type=merge -p '{"spec":{"bpfConnectTimeLoadBalancing":"Disabled","bpfHostNetworkedNATWithoutCTLB":"Disabled"}}'
```
