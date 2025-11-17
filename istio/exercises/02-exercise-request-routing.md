# Request Routing 

## Vorher (ohne request routing) 

  * Es werden alle Pods angezeigt, die das Label: app:reviews haben
  * D.h. jedesmal wenn ich die Seite öffne, wird eine andere Version angegezeigt (v1, v2 oder v3)
  * Service (selector: app:reviews)

```
kubectl -n bookinfo get svc reviews -o yaml
kubectl -n bookinfo get pods -l app=reviews --show-labels
```

## Übung (jetzt request - routing) 

**Voraussetzung:**

- Bookinfo-App läuft bereits im Namespace `bookinfo`
- Service Reviews ist deniert
- Es gibt 3 verschieden Pods an Reviews (v1, v2 und v3)
- Ingress/Gateway + `GATEWAY_URL` (IP: http://164.90.237.35/productpage aus der vorherigen Übung vorhanden

### 0. Vorbereitung

```bash
mkdir -p ~/manifests/requests
cd ~/manifests/requests
````

---

### 1. HTTPRoute: Alle Requests → `reviews-v1`

```bash
cat <<EOF > ~/manifests/requests/httproute-reviews-v1.yaml
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

kubectl apply -f ~/manifests/requests/httproute-reviews-v1.yaml
kubectl get httproute reviews -n bookinfo
```

---

### 2. HTTPRoute anpassen: User `jason` → `reviews-v2`, Rest → `reviews-v1`

```bash
cat <<EOF > ~/manifests/requests/httproute-reviews-jason-v2.yaml
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
  - matches:
    - headers:
      - name: end-user
        value: jason
    backendRefs:
    - name: reviews-v2
      port: 9080
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF

kubectl apply -f ~/manifests/requests/httproute-reviews-jason-v2.yaml
kubectl get httproute reviews -n bookinfo -o yaml
```

---

## 3. Testen im Browser

```bash
echo "$GATEWAY_URL"
# Beispiel: http://<IP>:<PORT>

# 1. Im Browser: $GATEWAY_URL/productpage aufrufen (nicht eingeloggt oder anderer User)
#    → Reviews ohne Sterne (v1)

# 2. Im Browser: als User "jason" einloggen
#    → Reviews mit Sternen (v2)
```

(Optional: Kurztest per curl, ohne Login-UI):

```bash
for i in {1..3}; do
  curl -s "$GATEWAY_URL/productpage" | grep -o "rating" || true
done

for i in {1..3}; do
  curl -s -H "end-user: jason" "$GATEWAY_URL/productpage" | grep -o "rating" || true
done
```

---

## 4. Aufräumen

```bash
kubectl delete -f ~/manifests/requests/httproute-reviews-v1.yaml --ignore-not-found
kubectl delete -f ~/manifests/requests/httproute-reviews-jason-v2.yaml --ignore-not-found
```


