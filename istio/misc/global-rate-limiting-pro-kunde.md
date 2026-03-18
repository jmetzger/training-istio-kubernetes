# Istio Per-Customer Rate Limiting mit JWT + Redis + Monitoring

Komplettes Setup für per-Kunde Rate Limiting im Istio Service Mesh.
Kunden authentifizieren sich per JWT, der `customer_id`-Claim wird als
Envoy-Descriptor an den externen Rate Limit Service gesendet. Monitoring
über StatsD → statsd-exporter → Prometheus/VictoriaMetrics → Grafana.

---

## Architektur

```
Client --[JWT]--> Gateway (istio-ingressgateway)
                      |
                      ├── 1. JWT validieren (RequestAuthentication)
                      ├── 2. Kein JWT? → 403 (AuthorizationPolicy)
                      ├── 3. customer_id Claim extrahieren
                      ├── 4. Descriptor {customer_id: "xyz"} an RLS senden (EnvoyFilter)
                      |         |
                      |         v
                      |   Rate Limit Service (gRPC :8081)
                      |     ├── StatsD ──► statsd-exporter (:9102) ──► Prometheus
                      |     |                                              |
                      |     v                                              v
                      |   Redis (:6379) — Counter pro Kunde           Grafana
                      |
                      ├── OK → weiter zum Backend
                      └── OVER_LIMIT → 429 Too Many Requests
                                       + x-envoy-ratelimited Header
```

### Begriffe

- **Descriptor**: Key-Value-Paar (z.B. `customer_id: "customer-a"`), das Envoy per gRPC an den Rate Limit Service schickt. Wird über `actions` im EnvoyFilter definiert. Mögliche Quellen: JWT Claims, HTTP Header, Client-IP, statische Werte.
- **Cluster-Patch**: Registriert den Rate Limit Service als Envoy-Cluster im Gateway mit einem kontrollierten Namen (`rate_limit_cluster`). Auch wenn der RLS im Mesh läuft und Istio automatisch einen Cluster anlegt, ist der explizite Patch der sauberere Weg — du kontrollierst den Cluster-Namen, der im HTTP Filter referenziert wird.
- **Domain**: Logischer Namespace im RLS, um Configs verschiedener Anwendungen zu trennen. Muss in EnvoyFilter und ConfigMap identisch sein.

---

## Komponenten

| # | Ressource | Namespace | Beschreibung |
|---|-----------|-----------|--------------|
| 1 | Namespace | — | `ratelimit` Namespace |
| 2 | Redis | ratelimit | Counter-Backend |
| 3 | ConfigMap ratelimit-config | ratelimit | Rate Limits pro Kunde |
| 4 | ConfigMap statsd-config | ratelimit | StatsD → Prometheus Mapping |
| 5 | Deployment + Service ratelimit | ratelimit | RLS + statsd-exporter Sidecar |
| 6 | PodMonitor | ratelimit | Prometheus Scrape Config (nur mit Operator) |
| 7 | RequestAuthentication | istio-system | JWT Validierung am Gateway |
| 8 | AuthorizationPolicy | istio-system | JWT erzwingen |
| 9 | EnvoyFilter | istio-system | Cluster-Patch + Rate Limit Filter + Descriptor |
| 10 | PeerAuthentication | ratelimit | Metrics-Port PERMISSIVE für Prometheus Scrape |
| 11 | Alerting Rules | ratelimit | PrometheusRule (Operator) oder ConfigMap (plain) |

---

## 1. Namespace (mit Sidecar Injection)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ratelimit
  labels:
    istio-injection: enabled
```

> **Mesh-Entscheidung**: In diesem Setup läuft der ratelimit-Namespace **im Mesh**.
> Das bedeutet: automatisches mTLS zwischen Gateway und RLS, bessere Observability
> (Istio Access Logs, Metriken für den gRPC-Traffic), und konsistentes Security-Modell
> wenn der Rest des Clusters STRICT mTLS fährt.
>
> Falls du den Namespace **nicht** im Mesh haben willst: `istio-injection: enabled`
> weglassen und Schritt 10 (PeerAuthentication) überspringen.

---

## 2. Redis

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: ratelimit
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ratelimit
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi
```

> **Produktion**: Redis Sentinel oder Redis Cluster verwenden.

---

## 3. Rate Limit Config (pro Kunde)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ratelimit-config
  namespace: ratelimit
data:
  config.yaml: |
    domain: my-api
    descriptors:
      # Kunde A: Premium - 1000 req/min
      - key: customer_id
        value: "customer-a"
        rate_limit:
          unit: minute
          requests_per_unit: 1000

      # Kunde B: Standard - 100 req/min
      - key: customer_id
        value: "customer-b"
        rate_limit:
          unit: minute
          requests_per_unit: 100

      # Kunde C: Free Tier - 10 req/min
      - key: customer_id
        value: "customer-c"
        rate_limit:
          unit: minute
          requests_per_unit: 10

      # Default: unbekannte Kunden
      - key: customer_id
        rate_limit:
          unit: minute
          requests_per_unit: 5
```

> **Hot Reload**: Der ratelimit-Service überwacht die Config-Dateien.
> ConfigMap-Änderungen werden automatisch übernommen (bis zu 60s Delay wegen kubelet sync).

> **Dynamische Limits**: Für viele Kunden / Self-Service kann ein Config-Generator
> (CronJob/Operator) die YAML aus einer Datenbank erzeugen und die ConfigMap updaten.
> Alternativ: Custom gRPC Rate Limit Service, der das `ShouldRateLimit`-Interface
> implementiert und Limits direkt aus einer DB liest.

---

## 4. StatsD-Exporter Mapping Config

Übersetzt die StatsD-Metriken des ratelimit-Service in Prometheus-Format
mit sinnvollen Labels (`domain`, `key1`, `key2`).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: statsd-config
  namespace: ratelimit
data:
  statsd.yaml: |
    mappings:
    # 2 Key-Levels: domain + key1 (z.B. customer_id_customer-a)
    - match: "ratelimit.service.rate_limit.*.*.near_limit"
      name: "ratelimit_service_rate_limit_near_limit"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
    - match: "ratelimit.service.rate_limit.*.*.over_limit"
      name: "ratelimit_service_rate_limit_over_limit"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
    - match: "ratelimit.service.rate_limit.*.*.total_hits"
      name: "ratelimit_service_rate_limit_total_hits"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
    - match: "ratelimit.service.rate_limit.*.*.within_limit"
      name: "ratelimit_service_rate_limit_within_limit"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"

    # 3 Key-Levels: domain + key1 + key2 (für verschachtelte Descriptors)
    - match: "ratelimit.service.rate_limit.*.*.*.near_limit"
      name: "ratelimit_service_rate_limit_near_limit"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
        key2: "$3"
    - match: "ratelimit.service.rate_limit.*.*.*.over_limit"
      name: "ratelimit_service_rate_limit_over_limit"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
        key2: "$3"
    - match: "ratelimit.service.rate_limit.*.*.*.total_hits"
      name: "ratelimit_service_rate_limit_total_hits"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
        key2: "$3"
    - match: "ratelimit.service.rate_limit.*.*.*.within_limit"
      name: "ratelimit_service_rate_limit_within_limit"
      observer_type: "histogram"
      labels:
        domain: "$1"
        key1: "$2"
        key2: "$3"

    # Config load Metriken
    - match: "ratelimit.service.config_load_success"
      name: "ratelimit_service_config_load_success"
      match_metric_type: counter
    - match: "ratelimit.service.config_load_error"
      name: "ratelimit_service_config_load_error"
      match_metric_type: counter

    # Alles andere droppen
    - match: "."
      match_type: "regex"
      action: "drop"
      name: "dropped"
```

> **Quelle**: https://github.com/envoyproxy/ratelimit/blob/main/examples/prom-statsd-exporter/conf.yaml

---

## 5. Rate Limit Service + StatsD-Exporter Sidecar

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ratelimit
  namespace: ratelimit
  labels:
    app: ratelimit
spec:
  ports:
  - port: 8080
    targetPort: 8080
    name: http
  - port: 8081
    targetPort: 8081
    name: grpc
  - port: 6070
    targetPort: 6070
    name: http-debug
  - port: 9102
    targetPort: 9102
    name: metrics
  selector:
    app: ratelimit
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratelimit
  namespace: ratelimit
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratelimit
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: ratelimit
      annotations:
        # Prometheus Scrape Annotations
        prometheus.io/scrape: "true"
        prometheus.io/port: "9102"
        prometheus.io/path: "/metrics"
        # Istio soll Metriken nicht mergen
        prometheus.istio.io/merge-metrics: "false"
    spec:
      containers:

      # --- Rate Limit Service ---
      - name: ratelimit
        # Gepinnter Commit-SHA statt :master (kein semantisches Versioning nach v1.4.0)
        # Aktuelle Tags: https://hub.docker.com/r/envoyproxy/ratelimit/tags
        image: envoyproxy/ratelimit:3fb70258
        command: ["/bin/ratelimit"]
        ports:
        - containerPort: 8080   # HTTP/REST
        - containerPort: 8081   # gRPC
        - containerPort: 6070   # Debug
        env:
        - name: LOG_LEVEL
          value: debug
        - name: LOG_FORMAT
          value: json
        - name: REDIS_SOCKET_TYPE
          value: tcp
        - name: REDIS_URL
          value: redis.ratelimit.svc.cluster.local:6379
        - name: RUNTIME_ROOT
          value: /data
        - name: RUNTIME_SUBDIRECTORY
          value: ratelimit
        - name: RUNTIME_WATCH_ROOT
          value: "false"
        - name: RUNTIME_IGNOREDOTFILES
          value: "true"
        # --- StatsD Konfiguration ---
        - name: USE_STATSD
          value: "true"
        - name: STATSD_HOST
          value: localhost        # Sidecar im selben Pod
        - name: STATSD_PORT
          value: "9125"
        - name: STATS_FLUSH_INTERVAL
          value: "10s"
        - name: GRPC_PORT
          value: "8081"
        volumeMounts:
        - name: config
          mountPath: /data/ratelimit/config
          readOnly: true
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi

      # --- StatsD Exporter Sidecar ---
      # Empfängt StatsD-Metriken vom ratelimit-Container auf localhost:9125
      # und exponiert sie als Prometheus-Metriken auf :9102/metrics
      - name: statsd-exporter
        image: prom/statsd-exporter:v0.27.2
        args:
        - "--statsd.mapping-config=/etc/statsd/statsd.yaml"
        - "--statsd.listen-udp=:9125"
        - "--web.listen-address=:9102"
        ports:
        - containerPort: 9125
          protocol: UDP
          name: statsd-udp
        - containerPort: 9102
          name: metrics
        volumeMounts:
        - name: statsd-config
          mountPath: /etc/statsd
          readOnly: true
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            memory: 128Mi

      volumes:
      - name: config
        configMap:
          name: ratelimit-config
      - name: statsd-config
        configMap:
          name: statsd-config
```

> **Image-Tag**: `envoyproxy/ratelimit` hat kein semantisches Versioning nach v1.4.0.
> Immer einen Commit-SHA pinnen (z.B. `3fb70258`), nicht `:master` verwenden.

---

## 6. Monitoring Scrape Config

> **Hinweis**: Das Prometheus aus dem Istio Demo-Stack (`istio/samples/addons`)
> ist ein **Plain Prometheus ohne Operator**. CRDs wie `PodMonitor` und
> `PrometheusRule` existieren dort nicht.

### Variante A: Plain Prometheus / Istio Demo (Standard)

Kein extra Manifest nötig. Das Deployment in Schritt 5 hat bereits die
Annotations gesetzt, die Plain Prometheus automatisch scraped:

```yaml
# Bereits im Deployment (Schritt 5) enthalten:
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9102"
  prometheus.io/path: "/metrics"
```

### Variante B: Prometheus Operator / VictoriaMetrics Operator

Nur wenn du den Prometheus Operator (z.B. kube-prometheus-stack) oder
VictoriaMetrics Operator nutzt:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: ratelimit
  namespace: ratelimit
  labels:
    release: prometheus   # Muss zu deinem Prometheus-Selector passen
spec:
  selector:
    matchLabels:
      app: ratelimit
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 15s
```

> **Nicht beides gleichzeitig** — Annotations + PodMonitor führt zu doppeltem Scraping.
> Bei Nutzung von PodMonitor die Annotations aus dem Deployment entfernen.

---

## 7. JWT Authentication (am Gateway)

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
    # Claim "customer_id" muss im JWT-Payload auf Top-Level vorhanden sein
    # z.B.: { "sub": "user123", "customer_id": "customer-a", "iss": "...", ... }
    # Verschachtelte Claims: path mit mehreren Keys angeben
```

> **Hinweis Gateway API**: Falls du ein Gateway API `Gateway`-Objekt statt des
> klassischen `istio-ingressgateway` nutzt, ersetze das Label mit:
> `istio.io/gateway-name: <name-deines-gateways>`

---

## 8. AuthorizationPolicy (JWT erzwingen)

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
```

---

## 9. EnvoyFilter: Rate Limit am Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ratelimit-filter
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
      # Für Gateway API stattdessen:
      # istio.io/gateway-name: <name-deines-gateways>
  configPatches:

  # ============================================================
  # CLUSTER-PATCH: Rate Limit Service als Envoy-Cluster registrieren
  # ============================================================
  # Auch wenn der RLS im Mesh läuft und Istio automatisch einen
  # Cluster anlegt, ist der explizite Patch sauberer: du kontrollierst
  # den Cluster-Namen, der im HTTP Filter referenziert wird.
  # Ohne diesen Patch: "unknown cluster 'rate_limit_cluster'" im Log.
  - applyTo: CLUSTER
    match:
      cluster:
        service: ratelimit.ratelimit.svc.cluster.local
    patch:
      operation: ADD
      value:
        name: rate_limit_cluster
        type: STRICT_DNS
        connect_timeout: 10s
        lb_policy: ROUND_ROBIN
        http2_protocol_options: {}        # gRPC braucht HTTP/2
        load_assignment:
          cluster_name: rate_limit_cluster
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: ratelimit.ratelimit.svc.cluster.local
                    port_value: 8081

  # ============================================================
  # HTTP FILTER: Rate Limit Filter in die Filter-Chain einfügen
  # ============================================================
  - applyTo: HTTP_FILTER
    match:
      context: GATEWAY
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.ratelimit
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
          domain: my-api                    # Muss mit ConfigMap übereinstimmen
          failure_mode_deny: false          # fail-open: bei RLS-Ausfall Requests durchlassen
          timeout: 0.25s
          rate_limit_service:
            grpc_service:
              envoy_grpc:
                cluster_name: rate_limit_cluster
            transport_api_version: V3

  # ============================================================
  # DESCRIPTOR: JWT Claim "customer_id" als Descriptor an RLS senden
  # ============================================================
  # Istio macht JWT-Claims automatisch als Envoy-Metadata verfügbar
  # unter dem Key "envoy.filters.http.jwt_authn".
  # Die Action extrahiert daraus den Claim "customer_id" und
  # sendet ihn als Descriptor {customer_id: "<wert>"} an den RLS.
  - applyTo: VIRTUAL_HOST
    match:
      context: GATEWAY
    patch:
      operation: MERGE
      value:
        rate_limits:
        - actions:
          - metadata:
              metadata_key:
                key: envoy.filters.http.jwt_authn
                path:
                - key: customer_id
              descriptor_key: customer_id
```

> **Ohne Cluster-Patch** bekommst du im Gateway-Log:
> `unknown cluster 'rate_limit_cluster'` und Rate Limiting funktioniert nicht.

> **`failure_mode_deny: false`** = fail-open (Requests durchlassen bei RLS-Ausfall).
> Auf `true` setzen für fail-closed (429 bei RLS-Ausfall).

> **Default Response**: Envoy gibt bei OVER_LIMIT **429 Too Many Requests** zurück
> plus den Header `x-envoy-ratelimited: true`. Anpassbar über `rate_limited_status`:
> ```yaml
> rate_limited_status:
>   code: 412    # statt 429 (nicht empfohlen)
> ```

---

## 10. PeerAuthentication (nötig weil ratelimit im Mesh)

Da der ratelimit-Namespace im Mesh läuft (`istio-injection: enabled`),
ist STRICT mTLS aktiv. Prometheus läuft aber typischerweise ohne Sidecar
und kann den Metrics-Port nicht über mTLS ansprechen. Deshalb Port 9102
auf PERMISSIVE setzen.

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: ratelimit
  namespace: ratelimit
spec:
  selector:
    matchLabels:
      app: ratelimit
  portLevelMtls:
    8081:
      mode: STRICT       # gRPC vom Gateway: mTLS (beide im Mesh)
    9102:
      mode: PERMISSIVE    # Prometheus hat keinen Sidecar, braucht Plain-Text-Zugriff
```

> **Ohne Mesh**: Wenn der ratelimit-Namespace nicht im Mesh ist
> (`istio-injection` nicht gesetzt), ist diese Ressource nicht nötig — überspringen.

---

## 11. Alerting

Die folgenden Alert-Rules funktionieren mit jedem Prometheus-kompatiblen System.
Die Einbindung unterscheidet sich je nach Setup.

### Alert Rules

```yaml
groups:
- name: ratelimit
  rules:
  # Alert wenn ein Kunde >50% over_limit hat
  - alert: CustomerRateLimitExceeded
    expr: |
      rate(ratelimit_service_rate_limit_over_limit_count{domain="my-api"}[5m])
      /
      rate(ratelimit_service_rate_limit_total_hits_count{domain="my-api"}[5m])
      > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kunde {{ $labels.key1 }} überschreitet Rate Limit"
      description: "Mehr als 50% der Requests von {{ $labels.key1 }} werden abgelehnt."

  # Alert wenn Rate Limit Service down
  - alert: RateLimitServiceDown
    expr: up{job="ratelimit"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Rate Limit Service ist nicht erreichbar"

  # Alert wenn Config-Load fehlschlägt
  - alert: RateLimitConfigError
    expr: increase(ratelimit_service_config_load_error[5m]) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Rate Limit Config konnte nicht geladen werden"
```

### Variante A: Plain Prometheus / Istio Demo

> **Hinweis**: Das Prometheus aus dem Istio Demo-Stack (`istio/samples/addons`)
> hat keinen Operator. `PrometheusRule` CRDs werden nicht unterstützt.

Die Rules als Datei in die Prometheus-ConfigMap einbinden:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: istio-system    # oder wo dein Prometheus läuft
data:
  alerting_rules.yml: |
    # <Alert Rules von oben hier einfügen>
```

Oder als separate Datei, referenziert in `prometheus.yml`:

```yaml
rule_files:
- /etc/prometheus/alerting_rules.yml
```

### Variante B: Prometheus Operator

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ratelimit-alerts
  namespace: ratelimit
spec:
  # <Alert Rules von oben hier einfügen>
```

---

## Deployment-Reihenfolge

```bash
# 1. Namespace
kubectl apply -f 01-namespace.yaml

# 2. Redis
kubectl apply -f 02-redis.yaml
kubectl -n ratelimit wait --for=condition=ready pod -l app=redis --timeout=60s

# 3. Configs (ratelimit + statsd mapping)
kubectl apply -f 03-ratelimit-config.yaml
kubectl apply -f 04-statsd-config.yaml

# 4. Rate Limit Service + StatsD Exporter
kubectl apply -f 05-ratelimit-deployment.yaml
kubectl -n ratelimit wait --for=condition=ready pod -l app=ratelimit --timeout=60s

# 5. Monitoring (nur bei Prometheus Operator, sonst überspringen - Annotations reichen)
# kubectl apply -f 06-podmonitor.yaml

# 6. PeerAuthentication (nötig weil ratelimit im Mesh)
kubectl apply -f 07-peer-authentication.yaml

# 7. JWT Auth + Policy
kubectl apply -f 08-request-authentication.yaml
kubectl apply -f 09-authorization-policy.yaml

# 8. EnvoyFilter (zuletzt!)
kubectl apply -f 10-envoyfilter-ratelimit.yaml

# 9. Alerting (PrometheusRule bei Operator, sonst ConfigMap - siehe Schritt 11)
# kubectl apply -f 11-prometheus-rules.yaml
```

---

## Testen

```bash
# Token für customer-a generieren
TOKEN_A="eyJhbGciOiJS..."  # JWT mit claim: customer_id: "customer-a"

# Einzelner Request - sollte 200 zurückgeben
curl -v -H "Authorization: Bearer $TOKEN_A" https://api.example.com/endpoint

# Ohne Token - sollte 403 zurückgeben (AuthorizationPolicy)
curl -v https://api.example.com/endpoint

# Last generieren - Rate Limit testen
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN_A" \
    https://api.example.com/endpoint
done
# Erwartete Response bei Rate Limit:
# HTTP/1.1 429 Too Many Requests
# x-envoy-ratelimited: true

# Metriken direkt vom statsd-exporter prüfen
kubectl -n ratelimit port-forward deploy/ratelimit 9102:9102 &
curl -s http://localhost:9102/metrics | grep ratelimit_service_rate_limit
# Erwarteter Output:
# ratelimit_service_rate_limit_total_hits_count{domain="my-api",key1="customer_id_customer-a"} 50
# ratelimit_service_rate_limit_within_limit_count{domain="my-api",key1="customer_id_customer-a"} 40
# ratelimit_service_rate_limit_over_limit_count{domain="my-api",key1="customer_id_customer-a"} 10

# Redis Counter direkt prüfen
kubectl -n ratelimit exec -it deploy/redis -- redis-cli KEYS '*'
```

---

## Grafana / Prometheus Queries

### Requests pro Kunde (Rate/Minute)

```promql
rate(ratelimit_service_rate_limit_total_hits_count{domain="my-api"}[1m]) * 60
```

### Over-Limit Requests pro Kunde

```promql
rate(ratelimit_service_rate_limit_over_limit_count{domain="my-api"}[1m]) * 60
```

### Near-Limit Warnungen (>80% verbraucht)

```promql
rate(ratelimit_service_rate_limit_near_limit_count{domain="my-api"}[1m]) > 0
```

### Prozent vom Limit verbraucht

Die Metrik weiß nicht, wie hoch das konfigurierte Limit ist.
Statische Zuordnung als Recording Rule oder Grafana-Variable nötig:

```promql
# customer-a: 1000 req/min
rate(ratelimit_service_rate_limit_total_hits_count{key1="customer_id_customer-a"}[1m]) * 60 / 1000 * 100

# customer-b: 100 req/min
rate(ratelimit_service_rate_limit_total_hits_count{key1="customer_id_customer-b"}[1m]) * 60 / 100 * 100
```

### Config Load Status (Healthcheck)

```promql
ratelimit_service_config_load_success   # Sollte > 0 sein
ratelimit_service_config_load_error     # Sollte 0 sein
```

---

## Debugging

```bash
# Rate Limit Service Logs
kubectl -n ratelimit logs -l app=ratelimit -c ratelimit -f

# StatsD Exporter Logs
kubectl -n ratelimit logs -l app=ratelimit -c statsd-exporter -f

# Gateway: Wurde der Cluster-Patch applied?
istioctl proxy-config cluster -n istio-system deploy/istio-ingressgateway | grep rate

# Gateway: Wurde der HTTP Filter applied?
istioctl proxy-config listener -n istio-system deploy/istio-ingressgateway -o json | grep ratelimit

# Gateway: Route Config mit Rate Limit Actions?
istioctl proxy-config route -n istio-system deploy/istio-ingressgateway -o json | grep rate

# Redis Counter prüfen
kubectl -n ratelimit exec -it deploy/redis -- redis-cli KEYS '*'
```

### Häufige Fehler

| Symptom | Ursache | Lösung |
|---------|---------|--------|
| `unknown cluster 'rate_limit_cluster'` | Cluster-Patch fehlt oder falsch | EnvoyFilter Cluster-Patch prüfen |
| Rate Limit greift nicht | Domain in EnvoyFilter ≠ ConfigMap | `domain: my-api` überall identisch |
| 500 statt 429 | RLS nicht erreichbar + `failure_mode_deny: true` | RLS Logs prüfen, auf `false` setzen |
| Metriken leer | `USE_STATSD=false` oder statsd-exporter nicht da | Env-Var + Sidecar prüfen |
| Metriken ohne Kunden-Labels | Mapping-Config fehlt oder falsch | statsd-config ConfigMap prüfen |

---

## Hinweise für Produktion

- **Mesh**: In diesem Setup läuft ratelimit im Mesh (`istio-injection: enabled`). Das gibt dir mTLS und Observability. Ohne Mesh: Label weglassen, Schritt 10 überspringen
- **`domain: my-api`** ist frei wählbar — muss aber in EnvoyFilter, ratelimit-Config und Grafana Queries identisch sein
- **StatsD → localhost**: ratelimit-Container sendet an `localhost:9125`, statsd-exporter Sidecar lauscht dort
- **`STATS_FLUSH_INTERVAL: 10s`**: Metriken werden alle 10s geflushed. Für Produktion 10-30s
- **Redis**: Für Produktion Redis Sentinel/Cluster verwenden
- **Replicas**: ratelimit-Service kann horizontal skaliert werden (State liegt in Redis)
- **Image-Tag**: Immer Commit-SHA pinnen, nicht `:master`
- **Istio-Upgrades**: EnvoyFilter ist eine unversioned API — bei jedem Istio/Envoy-Upgrade testen!
- **Metrik-Labels**: `key1` enthält `customer_id_<value>`. In Grafana filtern: `key1=~"customer_id_.*"`
- **JWT Claim**: `customer_id` muss auf Top-Level im JWT-Payload sein. Verschachtelte Claims über `path` mit mehreren Keys
- **Gateway API**: Label im EnvoyFilter anpassen: `istio.io/gateway-name: <n>` statt `istio: ingressgateway`, context bleibt `GATEWAY`
- **PeerAuthentication**: Nötig weil ratelimit im Mesh. Port 9102 PERMISSIVE damit Prometheus scrapen kann. Ohne Mesh nicht nötig
- **Monitoring**: Entweder PodMonitor (Prometheus Operator) ODER `prometheus.io/*` Annotations — nicht beides, sonst doppeltes Scraping
