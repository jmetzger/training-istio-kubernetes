# Übung: HTTPRoute mit LEAST_CONN Load Balancing (Bookinfo)

## Ziel

Traffic-Routing über die Gateway API (`HTTPRoute`) kombiniert mit Istio `DestinationRule` für LEAST_CONN Load Balancing – demonstriert an der Bookinfo-Anwendung. Sowohl für North-South (Gateway) als auch East-West (Service-zu-Service) Traffic.

## Voraussetzungen

- Kubernetes-Cluster mit Istio (inkl. Gateway API CRDs)
- `kubectl` und `istioctl` konfiguriert
- demo - app in bookinfo deployed 

---

## Schritt 1: Verteilung konfiguriert (für später in der Übung) 

```
mkdir -p ~/manifests/lb-least-conn
cd ~/manifests/lb-least-conn

# Die Service-Versionen anlegen
cp -a ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml bookinfo-versions.yaml
kubectl -n bookinfo apply -f .
```

## Schritt 2: Gateway erstellen (North-South)

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
```

```bash
kubectl apply -f gateway.yaml
```

---

## Schritt 3: Gateway-IP ermitteln und testen

```bash
export GATEWAY_IP=$(kubectl get gateway bookinfo-gateway -n bookinfo \
  -o jsonpath='{.status.addresses[0].value}')
echo "Gateway IP: $GATEWAY_IP"

# Bookinfo im Browser öffnen
echo "http://$GATEWAY_IP/productpage"
```

---

## Schritt 5: Reviews-Service auf 6 Replicas skalieren

```bash
kubectl scale deployment reviews-v1 reviews-v2 reviews-v3 \
  -n bookinfo --replicas=2
```

Ergebnis: 6 Reviews-Pods, je 2 pro Version.

---

## Schritt 6: DestinationRule mit LEAST_CONN für Reviews

```yaml
# destination-rule-reviews.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
  namespace: bookinfo
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

```bash
kubectl apply -f destination-rule-reviews.yaml
```

---

## Schritt 7: Verteilung beobachten (Baseline)

```bash
for i in $(seq 1 30); do
  curl -s http://$GATEWAY_IP/productpage | \
    grep -o 'glyphicon-star\|full-star' | head -1
done | sort | uniq -c
```

Alternativ über die Envoy-Access-Logs der Productpage:

```bash
kubectl logs -n bookinfo -l app=productpage -c istio-proxy --tail=50 | \
  grep "reviews" | awk '{print $5}' | sort | uniq -c
```

---

## Schritt 8: Last auf einem Reviews-Pod erzeugen

```bash
V1_POD=$(kubectl get pods -n bookinfo -l app=reviews,version=v1 \
  -o jsonpath='{.items[0].metadata.name}')
echo "Belaste Pod: $V1_POD"

kubectl exec -n bookinfo $V1_POD -c reviews -- \
  sh -c "dd if=/dev/zero of=/dev/null bs=1M &"
```

---

## Schritt 9: Verteilung unter Last vergleichen

### Mit LEAST_CONN (aktuelle Konfiguration)

```bash
echo "=== LEAST_CONN ==="
for i in $(seq 1 50); do
  curl -s http://$GATEWAY_IP/productpage > /dev/null &
done
wait

kubectl logs -n bookinfo -l app=productpage -c istio-proxy --tail=100 | \
  grep "reviews" | awk '{print $5}' | sort | uniq -c | sort -rn
```

> **Erwartung:** Der belastete v1-Pod bekommt weniger Requests.

### Umschalten auf ROUND_ROBIN

```bash
kubectl patch destinationrule reviews -n bookinfo --type merge \
  -p '{"spec":{"trafficPolicy":{"loadBalancer":{"simple":"ROUND_ROBIN"}}}}'

echo "=== ROUND_ROBIN ==="
for i in $(seq 1 50); do
  curl -s http://$GATEWAY_IP/productpage > /dev/null &
done
wait

kubectl logs -n bookinfo -l app=productpage -c istio-proxy --tail=100 | \
  grep "reviews" | awk '{print $5}' | sort | uniq -c | sort -rn
```

> **Erwartung:** Gleichmäßige Verteilung – auch der belastete Pod bekommt seinen Anteil.

### LEAST_CONN wieder aktivieren

```bash
kubectl patch destinationrule reviews -n bookinfo --type merge \
  -p '{"spec":{"trafficPolicy":{"loadBalancer":{"simple":"LEAST_CONN"}}}}'
```

---

## Schritt 10: Envoy-Konfiguration verifizieren

```bash
PP_POD=$(kubectl get pods -n bookinfo -l app=productpage \
  -o jsonpath='{.items[0].metadata.name}')

istioctl proxy-config cluster $PP_POD -n bookinfo | grep reviews
```

Erwartete Ausgabe:

```
reviews.bookinfo.svc.cluster.local   9080   -   outbound   LEAST_REQUEST
```

Detailliert:

```bash
istioctl proxy-config cluster $PP_POD -n bookinfo \
  --fqdn "reviews.bookinfo.svc.cluster.local" -o json | \
  jq '.[0].lbPolicy'
```

---

## Schritt 11: Last entfernen

```bash
kubectl exec -n bookinfo $V1_POD -c reviews -- sh -c "killall dd"
```

---

## Bonusaufgabe: HTTPRoute East-West Traffic Split + LEAST_CONN

Jetzt kombinieren wir beides: 80% Traffic an reviews-v3 (rote Sterne), 20% an reviews-v2 (schwarze Sterne). **Innerhalb** jeder Gruppe greift LEAST_CONN.

Seit der GAMMA-Initiative (Gateway API for Mesh Management and Administration) unterstützt Istio `HTTPRoute` auch für **East-West Traffic**. Dafür setzt man `parentRefs` auf einen **Service** statt auf einen Gateway.

### Zusätzliche Services pro Version anlegen

```yaml
# reviews-versioned-services.yaml
apiVersion: v1
kind: Service
metadata:
  name: reviews-v2
  namespace: bookinfo
spec:
  selector:
    app: reviews
    version: v2
  ports:
    - port: 9080
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-v3
  namespace: bookinfo
spec:
  selector:
    app: reviews
    version: v3
  ports:
    - port: 9080
```

```bash
kubectl apply -f reviews-versioned-services.yaml
```

### HTTPRoute mit Service als parentRef (East-West)

```yaml
# httproute-reviews-split.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews-split
  namespace: bookinfo
spec:
  parentRefs:
    - group: ""
      kind: Service
      name: reviews
      port: 9080
  rules:
    - backendRefs:
        - name: reviews-v3
          port: 9080
          weight: 80
        - name: reviews-v2
          port: 9080
          weight: 20
```

```bash
kubectl apply -f httproute-reviews-split.yaml
```

> **So funktioniert es:** Jeder Sidecar-Proxy, der `reviews:9080` aufruft (hier der Productpage-Sidecar), bekommt diese Routing-Regel injiziert. 80% der Requests landen bei reviews-v3, 20% bei reviews-v2.

### DestinationRules für die versionierten Services

```yaml
# destination-rules-versioned.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews-v2
  namespace: bookinfo
spec:
  host: reviews-v2
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews-v3
  namespace: bookinfo
spec:
  host: reviews-v3
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

```bash
kubectl apply -f destination-rules-versioned.yaml
```

### Verifizieren

```bash
for i in $(seq 1 50); do
  curl -s http://$GATEWAY_IP/productpage | \
    grep -oP '(full-star|glyphicon-star)' | head -1
done | sort | uniq -c
```

> **Erwartung:** ~80% rote Sterne (v3), ~20% schwarze Sterne (v2), keine Requests an v1.

---

## Aufräumen

```bash
kubectl delete namespace bookinfo
```

---

## Zusammenfassung

| Ebene | Ressource | parentRef | Aufgabe |
|---|---|---|---|
| **North-South** | `Gateway` + `HTTPRoute` | `kind: Gateway` | Externer Eintrittspunkt + Routing |
| **East-West** | `HTTPRoute` | `kind: Service` | Service-zu-Service Traffic Split |
| **LB-Algorithmus** | `DestinationRule` | – | LEAST_CONN, RANDOM, Consistent Hash |

**Merke:**
- `HTTPRoute` kann beides: North-South (`parentRef` → Gateway) **und** East-West (`parentRef` → Service)
- Für East-West setzt man `kind: Service` + `group: ""` in `parentRefs` – der Sidecar-Proxy aller Aufrufer wird konfiguriert
- `DestinationRule` definiert den LB-Algorithmus – unabhängig von der Routing-Ressource
- Kein VirtualService nötig – die Gateway API deckt beide Richtungen ab
