
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

- Bookinfo-App läuft bereits im Namespace `bookinfo` (ansonsten neu einrichten)
- Service Reviews ist definiert
- Es gibt 3 verschieden Pods an Reviews (v1, v2 und v3)
- Ingress/Gateway + `GATEWAY_URL` (IP: http://164.90.237.35/productpage aus der vorherigen Übung vorhanden

### 0 Vorbereitung

```bash
mkdir -p ~/manifests/requests
cd ~/manifests/requests

# Die Destinationen-Versionen anlegen
cp -a ~/istio/samples/bookinfo/networking/destination-rule-all.yaml destination-rule-all.yaml
kubectl -n bookinfo apply -f destination-rule-all.yaml 
```




---

### 1. VirtualService: Alle Requests → `reviews-v1`

```
nano reviews-v1.yaml 
```

```bash
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

```
kubectl -n bookinfo apply -f reviews-v1.yaml 
kubectl -n bookinfo get virtualservice reviews -n bookinfo
```

```
# Gateway IP ausfinden machen
kubectl -n istio-ingress get svc istio-ingress
GATEWAY_URL=<ip-aus-letzter-ausgabe>

```
# Anzeige im Browser - es ist immer die v1
http://$GATEWAY_URL/productpage
```


---

### 2. Adjust VirtualHost: User `jason` → `reviews-v2`, Rest → `reviews-v1`

```
cp -a ~/istio/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml virtual-service-reviews-test-v2.yaml
cat virtual-service-reviews-test-v2.yaml
```

```
kubectl -n bookinfo apply -f virtual-service-reviews-test-v2.yaml
kubectl -n bookinfo get vs reviews -n bookinfo -o yaml
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

---

## 4. Aufräumen

```bash
kubectl -n bookinfo delete vs reviews 
```

## Reference: 

  * https://istio.io/latest/docs/examples/bookinfo/#define-the-service-versions
