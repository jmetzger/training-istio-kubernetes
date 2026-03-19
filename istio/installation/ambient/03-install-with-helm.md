## Install Istio Ambient Mode with Helm

### Voraussetzung

- Helm-Charts `base`, `cni` und `istiod` bereits installiert (Version 1.29.1)
- Helm-Profile ambient: https://github.com/istio/istio/blob/master/manifests/helm-profiles/ambient.yaml

### Step 1: Gateway API CRDs (v1.4.0 - Standard)
```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

### Step 2: istiod und cni auf Ambient-Profil upgraden
```bash
# Zur Sicherheit, manchmal gibt es Probleme mit dem Mutating Webhook
kubectl delete validatingwebhookconfigurations istio-validator-istio-system

helm upgrade istiod istio/istiod -n istio-system --set profile=ambient --version 1.29.1 
helm upgrade istio-cni istio/cni -n istio-system --set profile=ambient --version 1.29.1 
```

### Step 3: ztunnel installieren
```bash
helm install ztunnel istio/ztunnel \
  --namespace istio-system \
  --version 1.29.1 \
  --wait
```
