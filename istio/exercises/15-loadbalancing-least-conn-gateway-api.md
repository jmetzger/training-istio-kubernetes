# Übung: LEAST_CONN Load Balancing mit HTTPRoute und DestinationRule

## Ziel

Den Unterschied zwischen LEAST_CONN und ROUND_ROBIN sichtbar machen. Dafür deployen wir einen Service mit schnellen und einem langsamen Pod. Bei parallelen Requests zeigt LEAST_CONN messbar besseres Verhalten.

## Voraussetzungen

- Kubernetes-Cluster mit Istio (inkl. Gateway API CRDs)
- `kubectl` und `istioctl` konfiguriert

---

## Schritt 1: Namespace vorbereiten

```bash
kubectl create namespace lb-demo
kubectl label namespace lb-demo istio-injection=enabled
```

---

## Schritt 2: Backend deployen (schnelle + langsamer Pod)

Alle Pods laufen hinter dem gleichen Service. Der einzige Unterschied: der langsame Pod wartet 2 Sekunden vor jeder Antwort.

```
cd 
mkdir -p manifests/lb-least/conn
cd manifests/lb-least/conn
```

```yaml
# backend.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-server
  namespace: lb-demo
data:
  server.py: |
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import time, os

    DELAY = float(os.environ.get('RESPONSE_DELAY', '0'))

    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            if DELAY > 0:
                time.sleep(DELAY)
            self.send_response(200)
            self.send_header('Content-Type', 'text/plain')
            self.end_headers()
            host = os.environ.get('HOSTNAME', 'unknown')
            self.wfile.write(f"{host} delay={DELAY}s\n".encode())
        def log_message(self, format, *args):
            pass

    HTTPServer(('', 8080), Handler).serve_forever()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-fast
  namespace: lb-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      variant: fast
  template:
    metadata:
      labels:
        app: backend
        variant: fast
    spec:
      containers:
        - name: server
          image: python:3-alpine
          command: ["python3", "/app/server.py"]
          env:
            - name: RESPONSE_DELAY
              value: "0"
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: server-script
              mountPath: /app
      volumes:
        - name: server-script
          configMap:
            name: backend-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-slow
  namespace: lb-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      variant: slow
  template:
    metadata:
      labels:
        app: backend
        variant: slow
    spec:
      containers:
        - name: server
          image: python:3-alpine
          command: ["python3", "/app/server.py"]
          env:
            - name: RESPONSE_DELAY
              value: "2"
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: server-script
              mountPath: /app
      volumes:
        - name: server-script
          configMap:
            name: backend-server
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: lb-demo
spec:
  selector:
    app: backend       # selektiert ALLE Pods – schnelle und langsame
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f backend.yaml
kubectl get pods -n lb-demo -w
```

Ergebnis: 3 Pods hinter einem Service – 2 schnelle (delay=0s), 1 langsamer (delay=2s).

---

## Schritt 3: Gateway + HTTPRoute

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: lb-gateway
  namespace: lb-demo
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-route
  namespace: lb-demo
spec:
  parentRefs:
    - name: lb-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /lb-test
      backendRefs:
        - name: backend
          port: 80
```

```bash
kubectl apply -f gateway.yaml

export GATEWAY_IP=$(kubectl get gateway lb-gateway -n lb-demo \
  -o jsonpath='{.status.addresses[0].value}')
echo "Gateway: http://$GATEWAY_IP/lb-test"
```

Schnelltest:

```bash
curl -s http://$GATEWAY_IP/lb-test
```

Erwartete Antwort: `backend-fast-xxxxx delay=0s` oder `backend-slow-xxxxx delay=2s`

---

## Schritt 4: DestinationRule mit LEAST_CONN

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: backend
  namespace: lb-demo
spec:
  host: backend
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
```

```bash
kubectl apply -f destination-rule.yaml
```

---

## Schritt 5: Envoy-Konfiguration prüfen

```bash
istioctl proxy-config cluster deploy/lb-gateway -n lb-demo | grep backend
```

Erwartete Ausgabe – `LEAST_REQUEST` in der Spalte LB POLICY:

```
backend.lb-demo.svc.cluster.local   80   -   outbound   LEAST_REQUEST
```

---

## Schritt 6: Test-Funktion definieren

Die Funktion sendet 30 parallele Requests (10 gleichzeitig) und zeigt:
- **Gesamtdauer** – wie lange alle Requests zusammen dauern
- **Verteilung** – wie viele Requests jeder Pod bekommen hat

```bash
test_lb() {
  local TMPDIR=$(mktemp -d)

  echo "--- Sende 30 Requests (10 parallel) ---"
  SECONDS=0
  seq 1 30 | xargs -P 10 -I{} \
    sh -c 'curl -s http://'"$GATEWAY_IP"'/lb-test > '"$TMPDIR"'/{}'
  echo "--- Gesamtdauer: ${SECONDS}s ---"

  echo "--- Verteilung pro Pod ---"
  cat "$TMPDIR"/* | sort | uniq -c | sort -rn

  rm -rf "$TMPDIR"
}
```

---

## Schritt 7: LEAST_CONN vs ROUND_ROBIN vergleichen

### Test mit LEAST_CONN

```bash
echo "=== LEAST_CONN ==="
test_lb
```

Erwartete Ausgabe:

```
--- Sende 30 Requests (10 parallel) ---
--- Gesamtdauer: 4s ---
--- Verteilung pro Pod ---
     14 backend-fast-abc123 delay=0s
     13 backend-fast-def456 delay=0s
      3 backend-slow-xyz789 delay=2s
```

> Der langsame Pod bekommt deutlich weniger Requests, weil seine Verbindungen 2s offen bleiben. LEAST_CONN weicht auf die schnellen Pods aus.

### Umschalten auf ROUND_ROBIN

```bash
kubectl patch destinationrule backend -n lb-demo --type merge \
  -p '{"spec":{"trafficPolicy":{"loadBalancer":{"simple":"ROUND_ROBIN"}}}}'

sleep 2

echo "=== ROUND_ROBIN ==="
test_lb
```

Erwartete Ausgabe:

```
--- Sende 30 Requests (10 parallel) ---
--- Gesamtdauer: 8s ---
--- Verteilung pro Pod ---
     10 backend-fast-abc123 delay=0s
     10 backend-fast-def456 delay=0s
     10 backend-slow-xyz789 delay=2s
```

> ROUND_ROBIN verteilt stur gleichmäßig – der langsame Pod bekommt genauso viele Requests. Die Gesamtdauer ist deutlich höher.

### Zurück auf LEAST_CONN

```bash
kubectl patch destinationrule backend -n lb-demo --type merge \
  -p '{"spec":{"trafficPolicy":{"loadBalancer":{"simple":"LEAST_CONN"}}}}'
```

---

## Schritt 8: Ergebnis

| Metrik | LEAST_CONN | ROUND_ROBIN |
|---|---|---|
| **Requests an langsamen Pod** | wenige (~3-5) | gleichmäßig (~10) |
| **Gesamtdauer (30 Req)** | ~4s | ~8s |
| **Prinzip** | Bevorzugt freie Pods | Verteilt stur reihum |

LEAST_CONN erkennt, dass der langsame Pod noch beschäftigt ist (offene Verbindung), und schickt neue Requests an die freien, schnellen Pods.

---

## Schritt 9: Wann welchen Algorithmus?

| Algorithmus | Envoy-Name | Wann sinnvoll? |
|---|---|---|
| **ROUND_ROBIN** | `ROUND_ROBIN` | Default. Gleichförmige, kurze Requests |
| **LEAST_CONN** | `LEAST_REQUEST` | Variable Latenzen, Long-Running Connections (gRPC, WebSocket) |
| **RANDOM** | `RANDOM` | Große Cluster, vermeidet Hot-Spots |
| **PASSTHROUGH** | – | Kein LB, direkt an Upstream-IP |

---

## Bonusaufgabe: HTTPRoute East-West mit Service parentRef

Seit der GAMMA-Initiative unterstützt Istio `HTTPRoute` auch für **East-West Traffic** (Service-zu-Service). Dafür setzt man `parentRefs` auf einen **Service** statt auf einen Gateway.

### Client-Pod deployen

```yaml
# client.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  namespace: lb-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
        - name: curl
          image: curlimages/curl:latest
          command: ["sleep", "infinity"]
```

```bash
kubectl apply -f client.yaml
```

### Zweiten Backend-Service anlegen

```yaml
# backend-fast-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-fast
  namespace: lb-demo
spec:
  selector:
    app: backend
    variant: fast
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f backend-fast-svc.yaml
```

### HTTPRoute: East-West Traffic Split

Alle Calls an `backend:80` werden zu 100% an die schnellen Pods umgeleitet:

```yaml
# httproute-east-west.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend-east-west
  namespace: lb-demo
spec:
  parentRefs:
    - group: ""
      kind: Service
      name: backend
      port: 80
  rules:
    - backendRefs:
        - name: backend-fast
          port: 80
          weight: 100
```

```bash
kubectl apply -f httproute-east-west.yaml
```

### Testen aus dem Client-Pod

```bash
CLIENT=$(kubectl get pods -n lb-demo -l app=client -o jsonpath='{.items[0].metadata.name}')

# Vorher: alle Pods (inkl. slow) erreichbar
# Nachher: nur noch fast-Pods antworten
for i in $(seq 1 10); do
  kubectl exec -n lb-demo $CLIENT -- curl -s http://backend/
done
```

> Alle Antworten kommen nur noch von `backend-fast` Pods – der langsame Pod ist per East-West HTTPRoute komplett aus dem Routing genommen.

---

## Aufräumen

```bash
kubectl delete namespace lb-demo
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
- Für East-West setzt man `kind: Service` + `group: ""` in `parentRefs`
- `DestinationRule` definiert den LB-Algorithmus – unabhängig von der Routing-Ressource
