# Istio Circuit Breaking – Übung (Sidecar Mode, Namespace `bookinfo`)

## Aufgabe des Circuit Breakers:

```
Der Circuit Breaker in Istio schützt deine Services davor, einen überlasteten oder fehlerhaften Backend-Service weiter mit Requests zu bombardieren.
```

## Wie ?

Du konfigurierst im `DestinationRule` unter `trafficPolicy.connectionPool` und `outlierDetection` zwei Dinge:

1. **Connection Pool Limits** (`connectionPool`) — begrenzt die Anzahl gleichzeitiger TCP-Connections und wartender HTTP-Requests. Wird das Limit überschritten, werden neue Requests sofort mit 503 abgelehnt statt endlos zu warten.

1. **Outlier Detection** (`outlierDetection`) — erkennt ungesunde Endpoints. Wenn ein Pod z.B. 5× hintereinander 5xx zurückgibt, wird er temporär aus dem Load-Balancing-Pool entfernt ("evicted"). Nach einer Pause darf er wieder rein.

## Voraussetzungen

* Istio ist installiert (Sidecar Mode).
* Namespace `bookinfo` ist mit automatischer Sidecar-Injection gelabelt:

```bash
kubectl label namespace bookinfo istio-injection=enabled --overwrite
```

* Das Istio-Repo liegt unter `~/istio` (Samples unter `~/istio/samples/...`).

---

## 1️⃣ Arbeitsverzeichnis anlegen

```bash
mkdir -p ~/manifests/circuit-breaker
cd ~/manifests/circuit-breaker
```

---

## 2️⃣ httpbin im Namespace `bookinfo` deployen

```bash
kubectl apply -n bookinfo -f ~/istio/samples/httpbin/httpbin.yaml
kubectl get pods -n bookinfo -l app=httpbin
```

Warte, bis der Pod `Running` ist.

---

## 3️⃣ DestinationRule mit Circuit Breaker anlegen

Erzeuge eine Manifestdatei im Übungsverzeichnis:

```
nano dr-httpbin-circuit-breaker.yaml
```

```
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin
  namespace: bookinfo
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```

> 🔐 **Hinweis (mTLS):**
> Wenn dein Mesh **strict mTLS** nutzt und du 503er bekommst, ergänze in `trafficPolicy` noch:
>
> ```yaml
>   tls:
>     mode: ISTIO_MUTUAL
> ```

Apply & prüfen:

```bash
kubectl apply -f dr-httpbin-circuit-breaker.yaml
kubectl get destinationrule httpbin -n bookinfo -o yaml
```

---

## 4️⃣ Fortio-Client im Mesh deployen

```bash
kubectl apply -n bookinfo -f ~/istio/samples/httpbin/sample-client/fortio-deploy.yaml
kubectl get pods -n bookinfo -l app=fortio
```

Exportiere den Pod-Namen:

```bash
export FORTIO_POD=$(kubectl get pods -n bookinfo -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
echo "$FORTIO_POD"
```

---

## 5️⃣ Sanity-Check: Ein einzelner Request

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio curl -quiet http://httpbin:8000/get
```

Erwartung: **HTTP 200 OK** mit JSON-Antwort.

---

## 6️⃣ Circuit Breaker „anrucken“ (2 Verbindungen)

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning \
  http://httpbin:8000/get
```

* `-c 2` → 2 gleichzeitige Verbindungen
* `-n 20` → 20 Requests

Erwartung: Die meisten Requests sind 200, einige ggf. 503.

---

## 7️⃣ Circuit Breaker deutlich auslösen (3 Verbindungen)

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning \
  http://httpbin:8000/get
```

Erwartung: **Deutlich mehr 503** (Circuit Breaking greift).

---

## 8️⃣ Envoy-Stats im Sidecar prüfen

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c istio-proxy -- \
  pilot-agent request GET stats | grep httpbin.bookinfo.svc.cluster.local | grep pending
```

Achte auf Werte wie `upstream_rq_pending_overflow` → zeigt an, wie viele Requests wegen Circuit Breaking abgewiesen wurden.

---

## 9️⃣ Aufräumen

```bash
# Circuit-Breaker-Regel entfernen
kubectl delete -f dr-httpbin-circuit-breaker.yaml

# Fortio-Client entfernen
kubectl delete -n bookinfo -f ~/istio/samples/httpbin/sample-client/fortio-deploy.yaml

# httpbin entfernen
kubectl delete -n bookinfo -f ~/istio/samples/httpbin/httpbin.yaml
```

---
