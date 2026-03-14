# Request Routing 

## Prerequisites

  * Bookinfo - Projekt aufgesetzt.

## Schritt 1: Vorbereitung: Status review-pods 

  * Status: alle Pods sind unter einem Service erreichbar

```
# Es gibt 3 verschieden review-pods (v1, v2, v3) 
kubectl -n bookinfo get pods --show-labels | grep review
```

```
# Ein Service zeigt auf alle pods (Alle versionen der Review - Pods)
kubectl -n bookinfo get svc | grep reviews  
```

## Schritt 2: Vorher (ohne request routing) 

  * Es werden alle Pods angezeigt, die das Label: app:reviews haben
  * D.h. jedesmal wenn ich die Seite öffne, wird eine andere Version angegezeigt (v1, v2 oder v3) - * d.h. es werden ganz normal die Services von Kubernetes verwendet **
  * Service (selector: app:reviews)

```
kubectl -n bookinfo get svc reviews -o yaml
kubectl -n bookinfo get pods -l app=reviews --show-labels
```

```
# Gateway wurde in der Übung vorher angelegt
# du findest so die IP des gateways raus 
kubectl -n bookinfo get gateway
```

```
GATEWAY_URL=<ip-aus-der-vorigen-Ausgabe-eintragen>
```

```
# Im Browser mehrmals ausführen
# Im Block mit den Reviews wechselt die Version 
$GATEWAY_URL/productpage
```


## Schritt 3: Übung (jetzt request - routing) 

**Voraussetzung:**

- Bookinfo-App läuft bereits im Namespace `bookinfo`
- Service Revies existiert 
- Es gibt 3 verschieden Pods an Reviews (v1, v2 und v3)
- Ingress/Gateway + `GATEWAY_URL` (IP: http://164.90.237.35/productpage aus der vorherigen Übung vorhanden

### 0. Vorbereitung

```bash
mkdir -p ~/manifests/requests
cd ~/manifests/requests

# Die Service-Versionen anlegen
cp -a ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml bookinfo-versions.yaml
kubectl -n bookinfo apply -f .
```

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

kubectl apply -f httproute-reviews-v1.yaml
kubectl -n bookinfo get httproute reviews -n bookinfo
```

```
# Vergleich mit allen httproutes
kubectl -n bookinfo get httproute
# booking for north - south traffik
```

```
# Anzeige im Browser - es ist immer die v1
http://164.90.237.35/productpage
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

kubectl apply -f httproute-reviews-jason-v2.yaml
kubectl -n bookinfo get httproute reviews -n bookinfo -o yaml
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
kubectl delete -f httproute-reviews-v1.yaml --ignore-not-found
kubectl delete -f httproute-reviews-jason-v2.yaml --ignore-not-found
```

## Reference: 

  * https://istio.io/latest/docs/examples/bookinfo/#define-the-service-versions
