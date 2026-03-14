# Service Mesh Debugging: Systematischer Workflow

## Ziel

Einen wiederholbaren Prozess vermitteln, mit dem Teilnehmer Performance-Probleme und Fehler in einem Istio Service Mesh systematisch eingrenzen und beheben können.

## Voraussetzungen

### Tools im Cluster

| Tool | Zweck | Installation (Demo/Training) |
|---|---|---|
| Prometheus + Grafana | Metrics & Dashboards | `kubectl apply -f samples/addons/prometheus.yaml` / `grafana.yaml` |
| Kiali | Service Graph & Health | `kubectl apply -f samples/addons/kiali.yaml` |
| Jaeger oder Tempo | Distributed Tracing | `kubectl apply -f samples/addons/jaeger.yaml` |

> **Hinweis:** Die Addon-Manifeste aus `samples/addons/` sind nur für Demo/Training geeignet. Für Produktion: eigene Deployments mit Persistence, Retention und HA.

### Demo-Applikation

Bookinfo-App mit aktiviertem Istio-Sidecar-Injection:

```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

---

## Der Debugging-Workflow

### Schritt 1: Symptom erkennen (Grafana / Prometheus)

Starte mit den **RED-Metrics** pro Service:

| Metrik | Frage | Istio-Metrik |
|---|---|---|
| **R**ate | Traffic-Volumen normal? | `istio_requests_total` |
| **E**rrors | 5xx-Rate erhöht? | `istio_requests_total{response_code=~"5.."}` |
| **D**uration | Latenz gestiegen? | `istio_request_duration_milliseconds` |

Beispiel-PromQL für p95-Latenz eines Service:

```promql
histogram_quantile(0.95,
  sum(rate(istio_request_duration_milliseconds_bucket{
    destination_service=~"reviews.*"
  }[5m])) by (le)
)
```

**Ergebnis:** Du weißt jetzt, *welcher Service* betroffen ist.

### Schritt 2: Wo im Call-Graph? (Kiali)

Kiali öffnen:

```bash
istioctl dashboard kiali
```

Im Service Graph prüfen:

- Welche Edges zeigen erhöhte Latenz (orange/rot)?
- Ist der betroffene Service selbst langsam, oder sein Downstream?
- Gibt es unerwartete Traffic-Pfade?

**Ergebnis:** Du weißt jetzt, *zwischen welchen Services* das Problem liegt.

### Schritt 3: Einzelnen Request verfolgen (Jaeger / Tempo)

Jaeger öffnen:

```bash
istioctl dashboard jaeger
```

Einen langsamen Trace auswählen und die Spans analysieren:

```
[Client] ──── 500ms ────> [productpage envoy] ── 5ms ──> [productpage app]
                                                              │
                                                              ├── 480ms ──> [reviews envoy] ── 3ms ──> [reviews app]
                                                              │                                            │
                                                              │                                            └── 450ms ──> [ratings]
                                                              └── 10ms ──> [details]
```

Hier siehst du pro Span:

- **Envoy-Proxy-Zeit:** Mesh-Overhead (mTLS, Routing, Retries)
- **App-Zeit:** Verarbeitungszeit der Anwendung selbst
- **Downstream-Zeit:** Wartezeit auf nachgelagerte Services

**Ergebnis:** Du weißt jetzt, ob das Problem im **Netzwerk**, in der **App** oder **downstream** liegt.

### Schritt 4: Ursache eingrenzen

| Befund | Mögliche Ursache | Nächster Schritt |
|---|---|---|
| Envoy-Proxy langsam | mTLS-Overhead, Retry-Storms, Rate Limiting | `istioctl proxy-config`, Envoy Stats prüfen |
| App selbst langsam | CPU-Limit, Memory-Pressure, langsame DB-Query | `kubectl top pod`, App-Profiling |
| Downstream langsam | Problem liegt beim Downstream-Service | Workflow dort wiederholen |
| Viele Retries sichtbar | Flapping Backend, falsche Timeout-Config | DestinationRule / VirtualService prüfen |
| Connection-Fehler | mTLS-Mismatch, PeerAuthentication falsch | `istioctl authn tls-check` |

### Schritt 5: Low-Level Debugging (Envoy / istioctl)

Wenn du tiefer gehen musst:

```bash
# Sync-Status aller Sidecars mit istiod prüfen
istioctl proxy-status

# Cluster-Config eines Pods inspizieren
istioctl proxy-config clusters <pod> -n <ns>

# Routen eines Pods inspizieren
istioctl proxy-config routes <pod> -n <ns>

# Listener eines Pods inspizieren
istioctl proxy-config listeners <pod> -n <ns>

# Envoy-Stats direkt abfragen
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET /stats

# Envoy-Config-Dump (vollständig)
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET /config_dump

# Istio-Analyse auf Fehlkonfigurationen
istioctl analyze -n <ns>
```

---

## Trainings-Lab: "Finde den Fehler"

### Setup: Latenz per Fault Injection erzeugen

Fault Injection auf den `reviews`-Service anwenden – 50% der Requests bekommen 3 Sekunden Delay:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews-delay
  namespace: default
spec:
  hosts:
  - reviews
  http:
  - fault:
      delay:
        percentage:
          value: 50
        fixedDelay: 3s
    route:
    - destination:
        host: reviews
```

```bash
kubectl apply -f reviews-delay.yaml
```

### Aufgabe für Teilnehmer

> "Die Bookinfo-App ist langsam. Findet heraus, welcher Service betroffen ist, wo die Latenz entsteht und was die Ursache ist."

### Erwarteter Lösungsweg

1. **Grafana:** p95-Latenz von `reviews` ist auf ~3s gestiegen
2. **Kiali:** Edge `productpage → reviews` zeigt erhöhte Latenz
3. **Jaeger:** Trace zeigt 3s Delay im `reviews`-Span, aber App-Zeit selbst ist minimal
4. **Erkenntnis:** Delay kommt nicht von der App – muss eine Mesh-Konfiguration sein
5. **Lösung finden:**
   ```bash
   kubectl get virtualservice -A
   kubectl describe vs reviews-delay
   ```
   → Fault Injection als Ursache identifiziert

### Varianten für weitere Labs

| Szenario | Injection | Lernziel |
|---|---|---|
| HTTP-Fehler | `fault.abort: 503, 30%` | Error-Rate Debugging |
| Timeout-Kette | Delay 5s + Timeout 2s in DestinationRule | Timeout-Konfiguration verstehen |
| Circuit Breaker | `outlierDetection` + Last erzeugen | Circuit Breaking analysieren |
| mTLS-Mismatch | `PeerAuthentication: STRICT` + Client ohne Sidecar | mTLS-Fehler debuggen |

---

## Zusammenfassung: Der Workflow auf einen Blick

```
Symptom erkennen          Grafana / Prometheus (RED-Metrics)
        │
        ▼
Ort eingrenzen            Kiali (Service Graph)
        │
        ▼
Request verfolgen         Jaeger / Tempo (Distributed Tracing)
        │
        ▼
Ursache klassifizieren    Netzwerk │ App │ Downstream │ Config
        │
        ▼
Low-Level Debugging       istioctl / Envoy Admin API
```

**Merksatz:** Von oben nach unten, von breit nach schmal – erst den Überblick, dann den Deep Dive.
