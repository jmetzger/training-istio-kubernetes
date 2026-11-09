# Istio (ambient) deinstallieren

Nur als Referenz. Reihenfolge einhalten: erst Namespaces aus dem Mesh nehmen, dann Istio entfernen.

## Step 1: Namespaces aus dem Mesh nehmen

```
# Beispiel: Namespace bookinfo
kubectl label namespace bookinfo istio.io/dataplane-mode-
kubectl label namespace bookinfo istio.io/use-waypoint-
istioctl waypoint delete --all -n bookinfo
```

Die Anwendungs-Pods müssen nicht neu gestartet werden (ambient hat keine Sidecars).

## Step 2: Istio entfernen

```
istioctl uninstall -y --purge
kubectl delete namespace istio-system

# Falls Istio-CRDs übrig bleiben (siehe Step 4), von Hand löschen:
kubectl get crd -o name | grep istio.io | xargs -r kubectl delete
```

## Step 3: Gateway API CRDs entfernen (optional)

Nur wenn nichts anderes im Cluster davon abhängt.

```
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

## Step 4: Überprüfen

```
kubectl get crd | grep -E 'istio.io|gateway.networking.k8s.io'
kubectl get pods -A | grep -E 'istio|ztunnel'
```

Beide Befehle sollten nichts ausgeben.
