# Istio Performance-Optimierung

## Überblick

Istio fügt durch die Envoy-Proxies einen gewissen Overhead hinzu – Latenz, CPU, Memory und I/O. Ziel der Performance-Optimierung ist es, diesen Overhead so gering wie möglich zu halten, ohne auf die Vorteile des Service Mesh zu verzichten.

---

## 1. Proxy-Latenz verstehen und messen

Jeder Envoy-Sidecar fügt Latenz hinzu – **typisch ~0,5ms pro Hop**, bei Ausreißern (z.B. TLS-Handshake, xDS-Update, hohe Last) bis zu 1–3ms. Ein Hop = ein Proxy-Durchgang. Da der Traffic sowohl den Sender- als auch den Empfänger-Sidecar passiert, kommen pro Service-Call ca. 1–2ms dazu (bei Ausreißern bis zu 6ms).

Bei langen Call-Chains summiert sich das:

```
Client → Frontend → Backend → Database
         +1-2ms     +1-2ms    +1-2ms
                                      = 3–6ms zusätzliche Latenz (typ.)
```

**Latenz messen mit Istio-Metriken:**

```promql
# p99 Latenz pro Service (server-side)
# p99 = 99. Perzentil: 99% aller Requests sind schneller als dieser Wert
histogram_quantile(0.99, 
  sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m])) 
  by (le, destination_service))

# Differenz zwischen client- und server-reported Latenz = Proxy-Overhead
histogram_quantile(0.99, rate(istio_request_duration_milliseconds_bucket{reporter="source"}[5m]))
-
histogram_quantile(0.99, rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
```

---

## 2. Protocol Detection vs. explizites appProtocol

Standardmäßig muss Envoy das Protokoll jeder eingehenden Verbindung erraten (Protocol Sniffing). Das kostet Zeit – Envoy wartet kurz, um die ersten Bytes zu inspizieren.

**Lösung:** Protokoll explizit angeben, dann entfällt die Sniffing-Phase:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api
spec:
  ports:
    - name: http           # Istio erkennt "http" im Port-Namen
      port: 80
      targetPort: 8080
      appProtocol: http    # explizit – spart Sniffing-Phase
```

Alternativ über die Port-Naming-Konvention: `http-`, `grpc-`, `tcp-` als Prefix.

> **Empfehlung:** Immer `appProtocol` setzen. Das ist die zuverlässigste Methode und spart pro Verbindung einige Millisekunden Initialisierung.

---

## 3. Access Logging tunen

Per Default loggt Envoy jeden einzelnen Request. Bei hohem RPS (Requests Per Second) erzeugt das erhebliche CPU- und I/O-Last.

### Wie Access Logging funktioniert

Access Logs sind die Logs der **Envoy-Proxies** – nicht die Anwendungslogs. Sie zeigen: wer hat wen aufgerufen, mit welchem Ergebnis. In `kubectl logs` sieht man beide, aber aus verschiedenen Containern:

```bash
# App-Logs
kubectl logs deploy/my-app -c my-app

# Envoy Access Logs
kubectl logs deploy/my-app -c istio-proxy
```

### Konfiguration

Access Logging wird über die **Telemetry-API** konfiguriert. Drei Aspekte:

- **Wohin** → `accessLogFile` in meshConfig (Dateipfad oder stdout) für den eingebauten Provider, oder `extensionProviders` für externe Ziele
- **Format** → `accessLogEncoding` in meshConfig (`TEXT` oder `JSON`)
- **Was** → `filter.expression` in der Telemetry-Resource

**Logging komplett deaktivieren:**

```yaml
meshConfig:
  accessLogFile: ""    # kein Ziel = nichts wird geloggt
```

**Nur Fehler loggen (empfohlen für Produktion):**

```yaml
# Schritt 1: Ziel und Format konfigurieren
meshConfig:
  accessLogFile: /dev/stdout    # wohin (stdout = sichtbar via kubectl logs)
  accessLogEncoding: JSON       # format
```

```yaml
# Schritt 2: Filter setzen – nur Fehler loggen
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: access-logging
  namespace: istio-system       # istio-system = gilt mesh-weit
spec:
  accessLogging:
    - providers:
        - name: envoy           # eingebauter Provider, nutzt accessLogFile
      filter:
        expression: "response.code >= 400"   # CEL-Expression: nur 4xx und 5xx
```

Die `filter.expression` ist eine CEL-Expression (Common Expression Language), die für jeden Request ausgewertet wird. Nur wenn sie `true` ergibt, wird geloggt. Weitere Beispiele:

- `"response.code >= 500"` – nur Server-Fehler
- `"response.code >= 400 || response.duration > 1000"` – Fehler oder langsame Requests (>1s)

### Mehrere Log-Ziele

Man kann zu verschiedenen Zielen mit unterschiedlichen Filtern loggen:

```yaml
# Externe Provider in meshConfig registrieren
meshConfig:
  accessLogFile: /dev/stdout
  extensionProviders:
    - name: otel-logging
      envoyOtelAls:
        service: otel-collector.monitoring.svc.cluster.local
        port: 4317
```

```yaml
# Verschiedene Filter pro Ziel
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: access-logging
  namespace: istio-system
spec:
  accessLogging:
    - providers:
        - name: envoy              # stdout: Client- und Server-Fehler
      filter:
        expression: "response.code >= 400"
    - providers:
        - name: otel-logging       # Collector: nur Server-Fehler
      filter:
        expression: "response.code >= 500"
```

`envoy` (stdout) ist immer verfügbar und muss nicht extra registriert werden. Alle anderen Provider müssen in `extensionProviders` definiert werden.

---

## 4. Tracing Sampling

Distributed Tracing erzeugt bei hohem RPS erheblichen Overhead – jeder Trace muss gesammelt, serialisiert und exportiert werden.

**Sampling-Rate anpassen:**

```yaml
meshConfig:
  defaultConfig:
    tracing:
      sampling: 1.0    # 1% der Requests – Default ist oft 100% in Demo-Setups!
```

| Umgebung | Empfohlenes Sampling |
|----------|---------------------|
| Development | 100% |
| Staging | 10–50% |
| Produktion (niedrig RPS) | 1–10% |
| Produktion (hoch RPS) | 0.1–1% |

> **Achtung:** Viele Istio-Demo-Installationen setzen Sampling auf 100%. Das ist ein häufiger Grund für unerklärlich hohen CPU-Verbrauch in Produktion.

---

## 5. Metrics Cardinality begrenzen

### Was ist Metrics Cardinality?

Istio-Metriken werden mit **Labels** (Key-Value-Paare) versehen. Jede einzigartige Kombination von Label-Werten erzeugt eine eigene Zeitreihe in Prometheus. Beispiel:

```
istio_requests_total{
  source_workload="frontend",
  destination_workload="backend-api",
  response_code="200",
  request_protocol="http",
  connection_security_policy="mutual_tls"
} 48273
```

Das ist ein Zähler: "Frontend hat Backend-API 48.273 mal mit HTTP aufgerufen und 200 zurückbekommen." Nicht jeder Request erzeugt eine neue Zeitreihe – aber jede **neue Kombination** von Label-Werten schon. Danach wird nur der Zähler hochgezählt:

```
...{source="frontend", destination="backend", response_code="200"} 48273  ← Zähler steigt
...{source="frontend", destination="backend", response_code="500"} 12     ← neue Kombination
...{source="checkout", destination="backend", response_code="200"} 9500   ← neue Kombination
```

### Warum ist die Anzahl der Labels entscheidend?

Jedes zusätzliche Label **multipliziert** die Anzahl der möglichen Kombinationen:

```
10 Sources × 10 Destinations                              = 100 Zeitreihen
10 Sources × 10 Destinations × 5 Response-Codes           = 500 Zeitreihen
10 Sources × 10 Destinations × 5 Codes × 2 Protocols      = 1.000 Zeitreihen
10 Sources × 10 Destinations × 5 Codes × 2 Prot. × 2 Sec. = 2.000 Zeitreihen
```

Entfernt man `request_protocol` und `connection_security_policy`, fällt man von 2.000 auf 500 Zeitreihen – ein Viertel. Und das pro Metrik. Bei 5 Istio-Metriken spart man 6.000 Zeitreihen.

### Hohe Cardinality = Problem für Prometheus

- Mehr Memory-Verbrauch (jede Zeitreihe lebt im RAM)
- Langsamere Queries
- Mehr Disk I/O

Bei 200 Services im Mesh können schnell Millionen von Zeitreihen entstehen.

### Labels entfernen

**Labels selektiv abschalten per Telemetry-API:**

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: reduce-metrics
  namespace: istio-system
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: ALL_METRICS
          tagOverrides:
            request_protocol:
              operation: REMOVE
            connection_security_policy:
              operation: REMOVE
```

> **Faustregel:** Labels entfernen, die man nicht aktiv in Dashboards oder Alerts abfragt. Wenn niemand je nach `connection_security_policy` filtert, bringt das Label keinen Nutzen, kostet aber Speicher.

---

## 6. Envoy Concurrency

Standardmäßig nutzt Envoy 2 Worker-Threads. Für High-Throughput-Workloads kann man das erhöhen:

```yaml
apiVersion: networking.istio.io/v1
kind: ProxyConfig
metadata:
  name: high-throughput
  namespace: backend
spec:
  concurrency: 4   # Anzahl Worker-Threads
```

> **Achtung:** Mehr Concurrency = mehr CPU-Verbrauch. Nur für Workloads mit hohem RPS sinnvoll.

---

## 7. mTLS-Overhead

Der TLS-Handshake und die Ver-/Entschlüsselung kosten CPU, aber in der Praxis ist der Overhead gering – typisch **< 0,5ms zusätzliche Latenz** und **~5–10% mehr CPU** auf dem Proxy.

Relevant wird es bei:

- Sehr hohem Throughput (>10.000 RPS pro Pod)
- Großen Payloads (Verschlüsselung skaliert mit Datenmenge)
- Ressourcen-knappe Umgebungen (Edge, IoT)

> **Empfehlung:** mTLS immer auf STRICT lassen. Der Sicherheitsgewinn überwiegt den minimalen Performance-Overhead bei weitem. Optimierung an anderen Stellen (Sidecar-Scope, Tracing, Logging) bringt deutlich mehr.

---

## 8. Connection Pooling: HTTP/2 vs. HTTP/1.1

Envoy nutzt standardmäßig Connection Pooling, aber das Verhalten unterscheidet sich je nach Protokoll:

- **HTTP/1.1:** Eine Connection pro Request (oder Keep-Alive mit begrenzten parallelen Requests)
- **HTTP/2:** Multiplexing – viele Requests über eine Connection

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: high-throughput-pool
spec:
  host: my-api
  trafficPolicy:
    connectionPool:
      http:
        h2UpgradePolicy: UPGRADE        # HTTP/1.1 → HTTP/2 upgraden
        http2MaxRequests: 1000           # max parallele Requests
        maxRequestsPerConnection: 0      # unbegrenzt (für HTTP/2)
      tcp:
        maxConnections: 100
        connectTimeout: 5s
```

> **Tipp:** Zwischen Sidecars im Mesh kann man bedenkenlos auf HTTP/2 upgraden, auch wenn die App selbst nur HTTP/1.1 spricht – Envoy übernimmt die Protokoll-Translation.

---

## 9. Custom Extensions: EnvoyFilter, Lua und WasmPlugin

Wenn man das Verhalten der Envoy-Proxies anpassen will (z.B. Header hinzufügen, Requests transformieren), gibt es drei Ansätze.

### 9.1 EnvoyFilter (native Config-Patches)

Man patcht direkt die Envoy-Konfiguration. Sehr performant, aber fragil – bricht leicht bei Istio-Upgrades, weil sich interne Strukturen ändern.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: add-timeout
  namespace: backend
spec:
  configPatches:
    - applyTo: ROUTE_CONFIGURATION
      patch:
        operation: MERGE
        value:
          request_timeout: 30s
```

### 9.2 Lua (via EnvoyFilter)

Lua-Scripts werden ebenfalls über ein `EnvoyFilter`-Objekt deployt – aber statt die Envoy-Config direkt zu patchen, injiziert man ein Script als HTTP-Filter. Das Script selbst ist stabil, das EnvoyFilter-Objekt drumherum kann bei Upgrades trotzdem brechen.

Zwei Einstiegspunkte: `envoy_on_request` (bevor der Request zum Backend geht) und `envoy_on_response` (bevor die Response zum Client geht).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: lua-add-header
  namespace: backend
spec:
  workloadSelector:
    labels:
      app: my-api
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
              subFilter:
                name: envoy.filters.http.router
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.lua
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
            inline_code: |
              function envoy_on_response(response_handle)
                response_handle:headers():remove("server")
                response_handle:headers():add("x-served-by", "mesh")
              end
```

### 9.3 WasmPlugin (empfohlener Weg)

Seit Istio die `WasmPlugin`-Resource hat, ist das der strategisch beste Ansatz. Vorteile:

- **Eigene Istio-API** – kein EnvoyFilter nötig, deutlich stabiler bei Upgrades
- **Sandbox** – Wasm läuft in einer VM, kann den Proxy nicht crashen
- **Portabel** – Wasm-Module lassen sich als OCI-Image verteilen
- **Mehrere Sprachen** – Go, Rust, C++ (via proxy-wasm SDK)

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: add-header
  namespace: backend
spec:
  selector:
    matchLabels:
      app: my-api
  url: oci://registry.example.com/my-wasm-plugin:v1
  phase: STATS               # wann im Filter-Chain: AUTHN, AUTHZ, STATS
  pluginConfig:
    header_name: "x-custom"
    header_value: "hello"
```

### Performance-Vergleich

| Ansatz | Latenz-Overhead | Memory | Upgrade-Stabilität |
|--------|----------------|--------|--------------------|
| **EnvoyFilter (native)** | Minimal | Gering | Fragil – bricht häufig |
| **Lua (via EnvoyFilter)** | Niedrig (~0,1–0,5ms) | Gering | Script stabil, EnvoyFilter-Wrapper fragil |
| **WasmPlugin** | Mittel (~0,5–2ms) | Höher (VM) | Stabil – eigene API |

### Empfehlung

- **WasmPlugin** als Standard für Custom Extensions – stabil, sicher, portabel
- **Lua** für schnelle, einfache Eingriffe (Header-Manipulation, kleine Transformationen)
- **EnvoyFilter (native Config-Patches)** nur als letztes Mittel, wenn es keine andere Option gibt

---

## 10. Best Practices Zusammenfassung

1. **appProtocol** immer explizit setzen – spart Protocol-Sniffing-Overhead
2. **Access Logging** auf Fehler beschränken oder deaktivieren – spart CPU/IO
3. **Tracing Sampling** in Produktion auf 0,1–1% setzen – häufigste Ursache für unerklärlich hohen CPU-Verbrauch
4. **Metrics Cardinality** bewusst begrenzen – Labels entfernen die nicht abgefragt werden
5. **Envoy Concurrency** nur bei High-Throughput-Workloads erhöhen
6. **mTLS auf STRICT lassen** – Overhead minimal, Sicherheitsgewinn groß
7. **HTTP/2 zwischen Sidecars** aktivieren bei vielen parallelen Requests
8. **WasmPlugin** als Standard für Custom Extensions – Lua für kleine Eingriffe, EnvoyFilter (native) vermeiden
