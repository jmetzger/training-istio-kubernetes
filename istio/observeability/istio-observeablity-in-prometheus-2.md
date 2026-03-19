# Übung: Istio Observability mit Prometheus & Grafana

## Lernziele

- Istio-Metriken verstehen, die in Ambient Mode von ztunnel (L4) und Waypoint Proxies (L7) erzeugt werden
- Prometheus & Grafana über die Istio-Samples deployen
- Die wichtigsten Grafana-Dashboards kennen und nutzen
- Unter Last das Verhalten der Bookinfo-App in Grafana beobachten

---

## 1. Hintergrund: Istio-Metriken im Ambient Mode

Im Ambient Mode gibt es **keine Sidecar-Proxies**. Stattdessen erzeugen zwei Komponenten die Metriken:

- **ztunnel** (läuft als DaemonSet auf jedem Node): Liefert **L4-Metriken** (TCP) für allen Traffic im Mesh – automatisch, ohne weitere Konfiguration.
- **Waypoint Proxy** (optionaler L7-Proxy pro Namespace/Service): Liefert **L7-Metriken** (HTTP Requests, Latenz, Response Codes) – nur wenn ein Waypoint deployt ist.

> **Wichtig:** Ohne Waypoint Proxy sieht man in Prometheus/Grafana nur TCP-Metriken. Für die vollen HTTP-Metriken (`istio_requests_total`, Latenz etc.) muss ein Waypoint Proxy aktiv sein.

### Wichtige Request-Metriken (L7 – benötigen Waypoint Proxy)

| Metrik | Beschreibung |
|---|---|
| `istio_requests_total` | Gesamtzahl aller Requests (Labels: `response_code`, `source_workload`, `destination_workload`) |
| `istio_request_duration_milliseconds` | Latenz als Histogramm (p50, p90, p99) |
| `istio_request_bytes` | Größe der eingehenden Requests |
| `istio_response_bytes` | Größe der Responses |

### TCP-Metriken (L4 – von ztunnel, immer verfügbar)

| Metrik | Beschreibung |
|---|---|
| `istio_tcp_sent_bytes_total` | Gesendete Bytes (TCP) |
| `istio_tcp_received_bytes_total` | Empfangene Bytes (TCP) |
| `istio_tcp_connections_opened_total` | Neue TCP-Verbindungen |
| `istio_tcp_connections_closed_total` | Geschlossene TCP-Verbindungen |

### Control-Plane-Metriken (Istiod)

| Metrik | Beschreibung |
|---|---|
| `pilot_xds_pushes` | Anzahl der Config-Pushes an ztunnel und Waypoint Proxies |
| `pilot_proxy_convergence_time` | Zeit bis eine Config-Änderung bei allen Proxies (ztunnel/Waypoint) angekommen ist |
| `pilot_conflict_inbound_listener` | Listener-Konflikte (Hinweis auf Fehlkonfiguration) |
| `galley_validation_passed` / `failed` | Validierung von Istio-Ressourcen |

---

## 2. Die Istio Grafana-Dashboards

Istio liefert fertige Grafana-Dashboards als JSON-Dateien unter `samples/addons/dashboards/`. Diese Dashboards decken alle Ebenen des Mesh ab:

### Mesh Dashboard

**Zweck:** Vogelperspektive auf das gesamte Service Mesh.

Zeigt globale Erfolgsrate (% erfolgreicher Requests), Gesamt-Throughput (Requests/s) und die Latenzverteilung über alle Services hinweg. Hier erkennt man sofort, ob es irgendwo im Mesh ein Problem gibt.

**Typische Fragen:** Wie gesund ist mein Mesh insgesamt? Gibt es gerade einen Anstieg an 5xx-Fehlern?

### Service Dashboard

**Zweck:** Drill-Down auf einen einzelnen Service.

Zeigt für einen ausgewählten Service: eingehende Erfolgsrate, Latenz (p50/p90/p99), Request-Volumen und die Aufschlüsselung nach aufrufenden Clients. Besonders nützlich um zu sehen, **wer** einen Service aufruft und wie performant die Antworten sind.

**Typische Fragen:** Wie schnell antwortet `productpage`? Welcher Client verursacht die meisten Fehler?

### Workload Dashboard

**Zweck:** Detailansicht pro Workload (Deployment/Pod).

Zeigt sowohl Inbound- als auch Outbound-Traffic eines Workloads, TCP-Verbindungsmetriken und Fehlerraten. Hiermit sieht man, ob ein bestimmtes Deployment Probleme macht.

**Typische Fragen:** Schickt mein `reviews-v3` Deployment fehlerhafte Antworten? Wie viel Outbound-Traffic erzeugt es?

### Istio Control Plane Dashboard

**Zweck:** Gesundheit von Istiod überwachen.

Zeigt xDS-Push-Raten, Konfigurationskonvergenz-Zeiten, Ressourcenverbrauch (CPU/Memory) von Istiod und eventuelle Fehler. Wichtig für den Betrieb des Mesh selbst.

**Typische Fragen:** Kommt die Konfiguration rechtzeitig bei den Proxies an? Ist Istiod überlastet?

#### Deep Dive: Konvergenzzeit & Istiod-Gesundheit

**Was ist die Konvergenzzeit?**

Die Metrik `pilot_proxy_convergence_time` misst die Zeit von "Istiod erkennt eine Config-Änderung" bis "alle betroffenen Proxies (ztunnel + Waypoint) haben das xDS-Update bestätigt". Konkret: Wenn du eine HTTPRoute- oder AuthorizationPolicy-Änderung anwendest, wie lange dauert es, bis diese Regel bei allen Proxies aktiv ist?

**Richtwerte für die Konvergenzzeit:**

- **< 500ms** – normal für kleine bis mittlere Cluster (< 100 Services)
- **< 1s** – akzeptabel für größere Cluster
- **> 2-5s** – problematisch, Config-Änderungen kommen verzögert an (z.B. neue Routing-Regeln greifen erst Sekunden später)

> In einem Training-Cluster mit Bookinfo liegt der Wert typischerweise im niedrigen Millisekundenbereich.

**Anzeichen für Istiod-Überlastung:**

| Metrik | Gesund | Achtung | Alarm |
|---|---|---|---|
| Convergence Time p99 | < 500ms | 1–5s | > 5s |
| xDS NACKs (`pilot_xds_pushes{type="nack"}`) | 0 | gelegentlich | dauerhaft > 0 |
| Push Duration p99 (`pilot_xds_push_time`) | < 100ms | 100ms–1s | > 1s |
| Istiod CPU | < 50% | 50–80% | > 80% sustained |

**Was bedeuten die einzelnen Werte?**

- **xDS NACKs:** Proxies lehnen die Config ab. Das ist ein klares Warnsignal – entweder fehlerhafte Config oder Istiod schickt inkonsistente Updates unter Last.
- **Push Duration:** Wie lange Istiod braucht, um ein xDS-Update zu berechnen und zu senden. Steigt dieser Wert, kommt Istiod nicht mehr hinterher.
- **Push Triggers (`pilot_push_triggers`):** Zeigt wie oft Pushes getriggert werden. Viele Triggers in kurzer Zeit (z.B. durch häufige Deployments oder rollende Updates) können Istiod überlasten.
- **Memory:** Wächst linear mit der Anzahl Services × Endpoints × Proxies. Bei hunderten Services können mehrere GB nötig sein.

**Nützliche PromQL-Queries für Istiod:**

```promql
# Konvergenzzeit p99 (letzte 5 Min)
histogram_quantile(0.99,
  sum(rate(pilot_proxy_convergence_time_bucket[5m])) by (le))

# NACK-Rate (sollte 0 sein!)
rate(pilot_xds_pushes{type="nack"}[5m])

# Push-Dauer p99
histogram_quantile(0.99,
  sum(rate(pilot_xds_push_time_bucket[5m])) by (le))
```

> **Praxis-Tipp:** Diese Werte sind in einem Training-Cluster immer im grünen Bereich. Die Referenzwerte sind aber wichtig für Teilnehmer, die das Wissen auf ihre Produktiv-Cluster übertragen wollen.

### Istio Performance Dashboard

**Zweck:** Ressourcenverbrauch der Mesh-Datenebene.

Im Ambient Mode zeigt dieses Dashboard den Ressourcenverbrauch von **ztunnel** (DaemonSet, läuft auf jedem Node) und **Waypoint Proxies**. Da es keine Sidecars gibt, entfällt der Per-Pod-Sidecar-Overhead. Stattdessen ist relevant, wie viel CPU/Memory ztunnel pro Node verbraucht und ob Waypoint Proxies skaliert werden müssen.

**Typische Fragen:** Wie viel Ressourcen verbraucht ztunnel auf meinen Nodes? Muss der Waypoint Proxy hochskaliert werden?

---

## 3. Hands-On: Setup

### Voraussetzungen

- Kubernetes-Cluster mit Istio im **Ambient Mode**
- Namespace `bookinfo` mit Ambient-Label: `istio.io/dataplane-mode=ambient`
- Waypoint Proxy für den Namespace deployt (für L7-Metriken)
- Bookinfo-App deployed im Namespace `bookinfo`
- Gateway API CRDs installiert und `Gateway`-Ressource konfiguriert
- `istioctl` installiert

> **Hinweis:** Ohne Waypoint Proxy sind nur L4/TCP-Metriken sichtbar. Für die volle Observability (HTTP-Metriken, Latenz, Response Codes) muss ein Waypoint aktiv sein:
>
> ```bash
> istioctl waypoint apply -n bookinfo --enroll-namespace
> ```

### Schritt 1: Addons deployen

Die Istio-Samples bringen Prometheus, Grafana, Kiali und Jaeger als fertige Manifeste mit:

```bash
kubectl apply -f samples/addons/
```

Warten bis alles läuft:

```bash
kubectl rollout status deployment/prometheus -n istio-system
kubectl rollout status deployment/grafana -n istio-system
```

### Schritt 2: Grafana öffnen

```bash
istioctl dashboard grafana
```

> Öffnet automatisch `http://localhost:3000` im Browser.

### Schritt 3: Dashboards prüfen

Navigiere in Grafana zu **Dashboards → Browse → Istio**. Du solltest folgende Dashboards sehen:

- Istio Mesh Dashboard
- Istio Service Dashboard
- Istio Workload Dashboard
- Istio Control Plane Dashboard
- Istio Performance Dashboard

> **Falls die Dashboards fehlen:** Die JSON-Dateien liegen in `samples/addons/dashboards/` und können über Grafana → Dashboards → Import manuell geladen werden.

---

## 4. Hands-On: Traffic erzeugen und beobachten

### Schritt 4: Basis-Traffic erzeugen

Öffne ein separates Terminal und erzeuge kontinuierlichen Traffic auf die Bookinfo-App:

```bash
# Gateway-URL ermitteln (Gateway API)
export GATEWAY_URL=$(kubectl get gateway bookinfo-gateway -n bookinfo \
  -o jsonpath='{.status.addresses[0].value}')

# Endlos-Schleife mit 1 Request/Sekunde
while true; do
  curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_URL/productpage
  sleep 1
done
```

### Schritt 5: Mesh Dashboard beobachten

1. Öffne das **Istio Mesh Dashboard**
2. Beobachte wie die **Global Request Volume** ansteigt
3. Prüfe die **Global Success Rate** – sie sollte bei ~100% liegen

### Schritt 6: Service Dashboard nutzen

1. Wechsle zum **Istio Service Dashboard**
2. Wähle im Dropdown den Service `productpage.bookinfo.svc.cluster.local`
3. Beobachte:
   - **Client Request Volume** – woher kommen die Requests?
   - **Client Success Rate** – alles grün?
   - **Client Request Duration** – wie sehen p50/p90/p99 aus?

### Schritt 7: Last erhöhen mit Fortio

Jetzt erzeugen wir echte Last, um die Dashboards unter Druck zu setzen:

```bash
# Fortio als Pod deployen im bookinfo Namespace (falls noch nicht vorhanden)
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml -n bookinfo

# Fortio-Pod ermitteln
export FORTIO_POD=$(kubectl get pods -n bookinfo -l app=fortio -o jsonpath='{.items[0].metadata.name}')

# 100 Requests mit 10 gleichzeitigen Connections
kubectl exec -n bookinfo $FORTIO_POD -- \
  fortio load -c 10 -qps 0 -n 100 \
  http://productpage.bookinfo:9080/productpage
```

Wechsle zurück zu Grafana und beobachte:
- **Latenz-Anstieg** im Service Dashboard
- **Request Volume Spike** im Mesh Dashboard
- **TCP Connections** im Workload Dashboard

---

## 5. Hands-On: Fehler provozieren und erkennen

### Schritt 8: Fault Injection aktivieren

Injiziere einen HTTP 500 Fehler für den `ratings`-Service:

> **Hinweis:** Fault Injection ist ein L7-Feature und benötigt einen **Waypoint Proxy**. VirtualService wird hier verwendet, da Gateway API (HTTPRoute) Fault Injection noch nicht nativ unterstützt.

```yaml
# fault-injection.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings-fault
  namespace: bookinfo
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percentage:
          value: 50
    route:
    - destination:
        host: ratings
```

```bash
kubectl apply -f fault-injection.yaml
```

### Schritt 9: Fehler in Grafana beobachten

1. Erzeuge weiterhin Traffic (Schleife aus Schritt 4)
2. Öffne das **Istio Service Dashboard** → Service: `ratings.bookinfo.svc.cluster.local`
3. Beobachte:
   - **Success Rate sinkt auf ~50%**
   - **Response Codes** zeigen Mix aus 200 und 500
4. Im **Mesh Dashboard** siehst Du die globale Erfolgsrate sinken

### Schritt 10: Prometheus direkt abfragen

Öffne Prometheus:

```bash
istioctl dashboard prometheus
```

Teste diese PromQL-Queries:

```promql
# Fehlerrate für ratings (5xx pro Sekunde)
rate(istio_requests_total{destination_service="ratings.bookinfo.svc.cluster.local", response_code=~"5.."}[1m])

# Erfolgsrate in Prozent
sum(rate(istio_requests_total{destination_service="ratings.bookinfo.svc.cluster.local", response_code!~"5.."}[1m]))
/
sum(rate(istio_requests_total{destination_service="ratings.bookinfo.svc.cluster.local"}[1m]))
* 100

# p99 Latenz für productpage
histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{destination_service="productpage.bookinfo.svc.cluster.local"}[1m])) by (le))
```

---

## 6. Aufräumen

```bash
kubectl delete virtualservice ratings-fault -n bookinfo
```

---

## Zusammenfassung

| Was | Wo sichtbar |
|---|---|
| Globale Mesh-Gesundheit | Mesh Dashboard |
| Service-spezifische Latenz & Fehler | Service Dashboard (benötigt Waypoint) |
| Workload In/Out-Traffic | Workload Dashboard |
| Istiod-Gesundheit | Control Plane Dashboard |
| ztunnel/Waypoint-Ressourcenverbrauch | Performance Dashboard |
| Eigene Queries | Prometheus UI (PromQL) |

**Kernaussage:** Im Ambient Mode liefert **ztunnel** automatisch L4-Metriken für allen Traffic. Für volle L7-Observability (HTTP-Metriken, Latenz, Response Codes) wird ein **Waypoint Proxy** benötigt – aber auch das funktioniert ohne eine einzige Zeile Code in der Anwendung zu ändern.
