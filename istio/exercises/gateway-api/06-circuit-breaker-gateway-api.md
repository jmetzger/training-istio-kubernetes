# Istio Circuit Breaking mit Gateway API – Übung (Ambient + Waypoint, Namespace `bookinfo`)

## Aufgabe des Circuit Breakers

```
Der Circuit Breaker in Istio schützt Deine Services davor, einen überlasteten oder fehlerhaften Backend-Service weiter mit Requests zu bombardieren.
```

## Warum `DestinationRule` und nicht `HTTPRoute`?

```
Die HTTPRoute (Gateway API) legt fest, WOHIN ein Request geht (Pfad, Header, Gewichte, Backend). Wie sich der Aufrufer gegenüber einem Backend verhält (Verbindungslimits, Ausreißer-Erkennung, Load-Balancing-Algorithmus, TLS), kennt die Gateway API nicht. Das bleibt in Istio Aufgabe der DestinationRule, die Du mit HTTPRoute und Waypoint einfach kombinierst.
```

## Wie ?

Du konfigurierst in der `DestinationRule` unter `trafficPolicy.connectionPool` und `outlierDetection` zwei Dinge:

1. **Connection Pool Limits** (`connectionPool`) — begrenzt die Anzahl gleichzeitiger TCP-Connections und wartender HTTP-Requests. Wird das Limit überschritten, werden neue Requests sofort mit 503 abgelehnt statt endlos zu warten.

1. **Outlier Detection** (`outlierDetection`) — erkennt ungesunde Endpoints. Wenn ein Pod z.B. 5× hintereinander 5xx zurückgibt, wird er temporär aus dem Load-Balancing-Pool entfernt ("evicted"). Nach einer Pause darf er wieder rein.

## Wo wird das durchgesetzt? (Ambient)

Im Ambient Mode gibt es keinen Sidecar. Die Regeln der `DestinationRule` setzt immer der **Envoy** durch, der den Request an das Backend schickt (ztunnel kann nur L4):

| Weg | Wer setzt den Circuit Breaker durch |
|---|---|
| Teil A: `fortio` → `httpbin` (im Mesh) | der **Waypoint** (`bookinfo` hat `istio.io/use-waypoint=waypoint`) |
| Teil B: Internet → Gateway → `httpbin` (Gateway API `HTTPRoute`) | der **Gateway-Pod** `bookinfo-gateway-istio` |

## Voraussetzungen

* Istio Ambient ist installiert, `bookinfo` ist nach `ambient/04-install-demo-app.md` eingerichtet:
  * Label `istio.io/dataplane-mode=ambient` und `istio.io/use-waypoint=waypoint`
  * Waypoint `waypoint` ist `PROGRAMMED`
  * Gateway `bookinfo-gateway` (Gateway API) existiert (nur für Teil B)

```bash
kubectl get ns bookinfo --show-labels
kubectl get gateway -n bookinfo
```

* Das Istio-Repo liegt unter `~/istio` (Samples unter `~/istio/samples/...`).

---

## 1️⃣ Arbeitsverzeichnis anlegen

```bash
mkdir -p ~/manifests/circuit-breaker-gateway-api
cd ~/manifests/circuit-breaker-gateway-api
```

---

## 2️⃣ httpbin und Fortio im Namespace `bookinfo` deployen

```bash
kubectl apply -n bookinfo -f ~/istio/samples/httpbin/httpbin.yaml
kubectl apply -n bookinfo -f ~/istio/samples/httpbin/sample-client/fortio-deploy.yaml
kubectl get pods -n bookinfo -l 'app in (httpbin,fortio)'
```

Warte, bis beide Pods `Running` sind (`1/1`, kein Sidecar-Container, das ist Ambient).

```bash
export FORTIO_POD=$(kubectl get pods -n bookinfo -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
echo "$FORTIO_POD"
```

---

## 3️⃣ Sanity-Check und Baseline (ohne Circuit Breaker)

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio curl -quiet http://httpbin:8000/get | head -12
```

Erwartung: **HTTP 200 OK**. Im Header steht `server: istio-envoy` — die Antwort kam über den **Waypoint**.

Last mit 3 Verbindungen, noch ohne Regel:

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning \
  http://httpbin:8000/get 2>&1 | grep "Code "
```

Erwartung: `Code 200 : 30 (100.0 %)`

---

## 4️⃣ DestinationRule mit Circuit Breaker anlegen

```
nano dr-httpbin-circuit-breaker.yaml
```

```yaml
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

  * Erklärung TCP/http
  * Request 1 → bekommt die eine Connection → wird verarbeitet
  * Request 2 → Connection belegt → geht in die Pending Queue (1 Platz)
  * Request 3 → Connection belegt, Queue voll → sofort 503

```
consecutive5xxErrors: 1 — Ein einziger 5xx-Fehler reicht, und der Endpoint fliegt raus. In Produktion eher 5–10, damit nicht ein einzelner Timeout sofort zur Eviction führt.

interval: 1s — Envoy prüft jede Sekunde, ob ein Endpoint auffällig ist. Das ist der Auswertungszyklus. Default wäre 10s.

baseEjectionTime: 3m — Ein evicted Endpoint bleibt 3 Minuten draußen, bevor er wieder in den Pool darf. Bei wiederholter Eviction multipliziert sich die Zeit — beim zweiten Mal 6 Minuten, beim dritten 9 Minuten usw.

maxEjectionPercent: 100 — Es dürfen alle Endpoints gleichzeitig evicted werden. Default ist 10%. In Produktion gefährlich — da willst du immer mindestens einen Endpoint behalten.
```

> 🔐 **Hinweis (mTLS):** Im Ambient Mode läuft der Verkehr zum Waypoint ohnehin über mTLS (HBONE). Du musst hier kein `tls: ISTIO_MUTUAL` ergänzen.

Apply & prüfen:

```bash
kubectl apply -f dr-httpbin-circuit-breaker.yaml
kubectl get destinationrule httpbin -n bookinfo -o yaml
```

---

## 5️⃣ Teil A: Circuit Breaker im Mesh auslösen (Waypoint)

Ein einzelner Request geht weiterhin durch:

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio curl -quiet http://httpbin:8000/get | head -3
```

2 Verbindungen („anrucken“):

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning \
  http://httpbin:8000/get 2>&1 | grep "Code "
```

* `-c 2` → 2 gleichzeitige Verbindungen
* `-n 20` → 20 Requests

Erwartung: alle oder fast alle 200.

3 Verbindungen (deutlich auslösen):

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning \
  http://httpbin:8000/get 2>&1 | grep "Code "
```

Erwartung: **Deutlich mehr 503** (Circuit Breaking greift). Beispiel: `Code 200 : 18`, `Code 503 : 12`. Die genaue Zahl schwankt von Lauf zu Lauf.

---

## 6️⃣ Metriken im Waypoint prüfen

Der Waypoint ist ein normaler Pod (`deploy/waypoint`), die Zähler holst Du dort:

```bash
kubectl exec -n bookinfo deploy/waypoint -- \
  pilot-agent request GET stats 2>/dev/null \
  | grep istio_requests_total | grep httpbin \
  | grep -o 'response_code\.[0-9]*\.grpc_response_status\.response_flags\.[A-Z-]*\|: [0-9]*$' | paste - -
```

Erwartung (Zahlen variieren):

```
response_code.200.grpc_response_status.response_flags.-	: 69
response_code.503.grpc_response_status.response_flags.UO	: 12
```

`response_flags.UO` = **U**pstream **O**verflow: Envoy hat den Request wegen des Circuit Breakers sofort abgewiesen. Die Zahl hinter `UO` entspricht den 503ern aus Fortio.

> ℹ️ Im Sidecar Mode zeigt `upstream_rq_pending_overflow` im `istio-proxy` des Clients den Zähler. Der Waypoint exportiert diesen Cluster-Zähler nicht, deshalb nehmen wir hier die Metrik mit dem Flag `UO`.

---

## 7️⃣ Teil B: Circuit Breaker auch am Gateway (HTTPRoute)

Jetzt dieselbe `DestinationRule` auf dem Weg **Gateway → httpbin**. Dafür veröffentlichen wir `httpbin` über eine `HTTPRoute` am vorhandenen `bookinfo-gateway`.

```
nano httproute-httpbin.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin
  namespace: bookinfo
spec:
  parentRefs:
  - name: bookinfo-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /httpbin
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: httpbin
      port: 8000
```

* `/httpbin/get` wird zu `/get` umgeschrieben (`URLRewrite`), httpbin kennt den Prefix nicht.
* Die Circuit-Breaker-Regeln stehen **nicht** in der HTTPRoute, sie kommen aus der DestinationRule `httpbin` (Schritt 4).

```bash
kubectl apply -f httproute-httpbin.yaml
kubectl get httproute httpbin -n bookinfo
```

Ein Request von außen:

```bash
GATEWAY_IP=$(kubectl get gateway bookinfo-gateway -n bookinfo -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY_IP
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP/httpbin/get
```

Erwartung: `200`.

> ⚠️ Hat der Listener des Gateways einen `hostname` (z. B. nach Übung 17: `bookinfo.tln<N>.do.t3isp.de`), brauchst Du den Host-Header: `curl -H "Host: bookinfo.tln<N>.do.t3isp.de" ...` bzw. direkt die URL mit dem Hostnamen.

Last erzeugen: mit Fortio **im Cluster** gegen den Gateway-Service, damit die Verbindungen parallel ankommen:

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning \
  http://bookinfo-gateway-istio.bookinfo.svc.cluster.local/httpbin/get 2>&1 | grep "Code "
```

(Mit Hostname am Listener zusätzlich `-H "Host: bookinfo.tln<N>.do.t3isp.de"` vor der URL.)

Erwartung: wieder 200 **und** 503. Die 503 kommen diesmal vom **Gateway-Pod**:

```bash
kubectl exec -n bookinfo deploy/bookinfo-gateway-istio -- \
  pilot-agent request GET stats 2>/dev/null \
  | grep istio_requests_total | grep httpbin \
  | grep -o 'response_code\.[0-9]*\.grpc_response_status\.response_flags\.[A-Z-]*\|: [0-9]*$' | paste - -
```

Wieder `response_flags.UO` bei `503`.

---

## 8️⃣ Aufräumen

```bash
# HTTPRoute entfernen (Teil B)
kubectl delete -f httproute-httpbin.yaml

# Circuit-Breaker-Regel entfernen
kubectl delete -f dr-httpbin-circuit-breaker.yaml

# Fortio-Client entfernen
kubectl delete -n bookinfo -f ~/istio/samples/httpbin/sample-client/fortio-deploy.yaml

# httpbin entfernen
kubectl delete -n bookinfo -f ~/istio/samples/httpbin/httpbin.yaml
```

---
