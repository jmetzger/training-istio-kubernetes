# Traffic Shifting


### 0. Vorbereitung

```bash
mkdir -p ~/manifests/traffic-shifting
cd ~/manifests/traffic-shifting 

# Die Service-Versionen anlegen
cp -a ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml bookinfo-versions.yaml
kubectl -n bookinfo apply -f .
```
### 1. 100% Traffic -> reviews.v1 

```
cat <<'EOF' > ~/manifests/traffic-shifting/route-reviews-v1.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF
```

```
kubectl apply -n bookinfo -f route-reviews-v1.yaml
kubectl get httproute -n bookinfo reviews -o yaml | head -n 30
```

### 2. Testen 

```
# Seite öffnen
http://<deine-ip>/productpage

# Egal wie oft du die Seite lädst, es bleibt immer v1
```



## Reference:

 * https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/
