# Skalierung von Istio

## Überblick

Istio besteht aus einer **Control Plane** (istiod) und einer **Data Plane** (Envoy-Proxies). Beide müssen unabhängig voneinander skaliert werden. Die größte Herausforderung bei wachsenden Meshes ist nicht CPU oder RAM, sondern die **xDS-Konfigurationsverteilung** – also die Menge an Routing-Regeln, Endpoints und Policies, die istiod an jeden einzelnen Proxy pushen muss.

---

## 1. Control Plane Skalierung (istiod)

### 1.1 Horizontale Skalierung

istiod ist stateless bezüglich der Mesh-Konfiguration (sie kommt aus Kubernetes-API-Objekten). Mehrere Replicas können parallel laufen.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istiod
  namespace: istio-system
spec:
  replicas: 3
```

Empfehlungen:

- **Kleine Meshes** (< 100 Pods): 1 Replica reicht
- **Mittlere Meshes** (100–500 Pods): 2–3 Replicas
- **Große Meshes** (500+ Pods): 3+ Replicas mit HPA

### 1.2 HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: istiod
  namespace: istio-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: istiod
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

### 1.3 Ressourcen-Limits

```yaml
resources:
  requests:
    cpu: 500m
    memory: 2Gi
  limits:
    cpu: "2"
    memory: 4Gi
```

> **Faustregel:** istiod braucht vor allem Memory. Jeder Service/Endpoint verbraucht Speicher für die interne Konfigurationsdatenbank. Bei 1000+ Services sollte man mit 4–8 Gi rechnen.

---

## 2. xDS-Konfigurationsumfang reduzieren

Dies ist der **wichtigste Hebel** für Istio-Skalierung. Standardmäßig bekommt jeder Envoy-Proxy die Konfiguration für **alle** Services im Mesh – auch für Services, mit denen er nie kommuniziert.

### 2.1 Sidecar-Resource

Die `Sidecar`-Resource beschränkt den Sichtbarkeitsbereich eines Workloads auf nur die Services, die er tatsächlich braucht.

**Mesh-weiter Default** – nur eigenen Namespace + istio-system sehen:

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: istio-system   # mesh-weit wenn in istio-system
spec:
  egress:
    - hosts:
        - "./*"               # eigener Namespace
        - "istio-system/*"    # Istio-interne Services
```

**Namespace-spezifisch** – z.B. ein Frontend, das nur das Backend-API braucht:

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: frontend
spec:
  egress:
    - hosts:
        - "./*"
        - "istio-system/*"
        - "backend/*"
```

**Auswirkung:** Reduziert die Anzahl der Cluster/Endpoints/Routes in der Envoy-Konfiguration drastisch. Bei einem Mesh mit 200 Services sieht ein Frontend-Pod statt 200 nur noch 10–20 Service-Konfigurationen.

### 2.2 Discovery Selectors (meshConfig)

Discovery Selectors steuern, welche **Namespaces** istiod überhaupt überwacht. Standardmäßig watched istiod alle Namespaces im Cluster – auch solche, die gar nicht zum Mesh gehören (z.B. `monitoring`, `logging`, `ci-cd`).

Mit Discovery Selectors sagt man: "istiod, kümmere dich nur um Namespaces mit bestimmten Labels." Alles andere wird komplett ignoriert – keine Watches auf der Kubernetes-API, keine Services/Endpoints in der internen Datenbank, kein xDS-Push dafür.

**IstioOperator-Variante:**

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    discoverySelectors:
      - matchLabels:
          istio-discovery: enabled
```

**Installation via Helm Chart:**

```bash
helm install istio-base istio/base -n istio-system --create-namespace

helm install istiod istio/istiod -n istio-system \
  --set meshConfig.discoverySelectors[0].matchLabels.istio-discovery=enabled
```

Oder sauberer über eine `values.yaml`:

```yaml
# values-istiod.yaml
meshConfig:
  discoverySelectors:
    - matchLabels:
        istio-discovery: enabled
    - matchExpressions:
        - key: env
          operator: In
          values: ["production", "staging"]
```

```bash
helm install istiod istio/istiod -n istio-system -f values-istiod.yaml
```

> **Hinweis:** Das `matchExpressions`-Beispiel zeigt, dass man auch Set-basierte Selektoren nutzen kann – z.B. um mehrere Environments gleichzeitig einzuschließen.

**Namespaces labeln:**

```bash
kubectl label namespace frontend istio-discovery=enabled
kubectl label namespace backend istio-discovery=enabled
kubectl label namespace database istio-discovery=enabled
```

Nur Namespaces mit dem Label `istio-discovery: enabled` werden von istiod beobachtet. Das reduziert:

- Watch-Last auf der Kubernetes-API
- Memory-Verbrauch von istiod
- xDS-Push-Umfang

**Unterschied zur Sidecar-Resource:**

| | Discovery Selectors | Sidecar-Resource |
|---|---|---|
| **Ebene** | istiod – was wird überhaupt beobachtet | Pro Proxy – was bekommt er davon zu sehen |
| **Wirkung** | Namespace existiert für istiod nicht | Namespace existiert, wird aber gefiltert |
| **Spart** | API-Watches, Memory in istiod | xDS-Config-Größe pro Proxy |

Am besten kombiniert man beides: Discovery Selectors schließen irrelevante Namespaces komplett aus, Sidecar-Resources filtern innerhalb der relevanten Namespaces die Sichtbarkeit pro Workload.

### 2.3 exportTo

Auf Ressourcen-Ebene kann man die Sichtbarkeit einschränken:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: my-api
  namespace: backend
spec:
  exportTo:
    - "."        # nur im eigenen Namespace sichtbar
    - "frontend" # und im frontend Namespace
  hosts:
    - my-api
  http:
    - route:
        - destination:
            host: my-api
```

`exportTo` funktioniert auf: `VirtualService`, `DestinationRule`, `ServiceEntry`.

---

## 3. Data Plane Skalierung

### 3.1 Sidecar-Ressourcen pro Workload

Die Envoy-Proxies verbrauchen per Default relativ wenig, aber bei High-Traffic-Workloads muss man die Ressourcen anpassen:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/proxyCPU: "200m"
        sidecar.istio.io/proxyMemory: "256Mi"
        sidecar.istio.io/proxyCPULimit: "1"
        sidecar.istio.io/proxyMemoryLimit: "512Mi"
```

### 3.2 Envoy Concurrency

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

### 3.3 Ambient Mode als Skalierungsalternative

Im **Ambient Mode** entfällt der Sidecar-Proxy pro Pod komplett. Stattdessen:

- **ztunnel** (Node-Level): Übernimmt mTLS und L4-Routing – ein DaemonSet-Pod pro Node statt ein Sidecar pro Pod
- **Waypoint Proxy** (Namespace/Service-Level): Wird nur bei Bedarf für L7-Policies deployt

**Vorteile für Skalierung:**

| Aspekt | Sidecar-Mode | Ambient Mode |
|--------|-------------|--------------|
| Proxy-Anzahl | 1 pro Pod | 1 ztunnel pro Node + Waypoints |
| Memory-Overhead | ~50–100 MB pro Pod | ~50 MB pro Node |
| xDS-Konfiguration | Jeder Sidecar braucht Konfig | Nur ztunnel + Waypoints |
| Startup-Latenz | Sidecar muss mit Pod starten | Kein Sidecar-Init nötig |

Bei einem Cluster mit 100 Nodes und 2000 Pods:
- **Sidecar-Mode:** 2000 Envoy-Proxies, ~100–200 GB Memory-Overhead
- **Ambient Mode:** 100 ztunnel + wenige Waypoints, ~5–10 GB Memory-Overhead

Aktivierung:

```bash
kubectl label namespace my-app istio.io/dataplane-mode=ambient
```

---

## 4. Monitoring der Skalierung

### 4.1 Wichtige istiod-Metriken

| Metrik | Bedeutung | Alarmschwelle |
|--------|-----------|---------------|
| `pilot_xds_pushes` | Anzahl xDS-Pushes | Spikes = häufige Config-Änderungen |
| `pilot_xds_push_time` | Dauer eines Push-Zyklus | > 5s problematisch |
| `pilot_proxy_convergence_time` | Zeit bis alle Proxies aktuell | > 10s kritisch |
| `pilot_virt_services` | Anzahl VirtualServices | Wachstum beobachten |
| `pilot_services` | Anzahl Services im Mesh | Wachstum beobachten |
| `pilot_xds_config_size_bytes` | Größe der xDS-Config | Sidecar-Scope prüfen wenn hoch |

### 4.2 Envoy-Metriken pro Proxy

```bash
# Config-Dump eines Sidecars ansehen
kubectl exec -it deploy/my-app -c istio-proxy -- \
  pilot-agent request GET config_dump | jq '.configs | length'

# Cluster-Anzahl (= bekannte Upstream-Services)
kubectl exec -it deploy/my-app -c istio-proxy -- \
  pilot-agent request GET clusters | wc -l
```

Wenn ein einzelner Sidecar hunderte Cluster hat, ist die `Sidecar`-Resource nicht richtig konfiguriert.

### 4.3 Grafana Dashboards

Istio liefert vorkonfigurierte Dashboards:

- **Istio Control Plane Dashboard**: istiod CPU, Memory, Push-Zeiten, xDS-Connections
- **Istio Mesh Dashboard**: Überblick über alle Services und deren Traffic
- **Istio Performance Dashboard**: Proxy-Latenz und Ressourcenverbrauch

---

## 5. Best Practices Zusammenfassung

1. **Sidecar-Resource ist Pflicht** in produktiven Meshes – mesh-weiten Default setzen, dann pro Namespace verfeinern:

   ```yaml
   # Mesh-weiter Default: restriktiver Ausgangspunkt
   apiVersion: networking.istio.io/v1
   kind: Sidecar
   metadata:
     name: default
     namespace: istio-system      # istio-system = gilt mesh-weit
   spec:
     egress:
       - hosts:
           - "./*"                 # nur eigener Namespace
           - "istio-system/*"      # + Istio-interne Services
   ```

   Dann pro Namespace verfeinern, wo nötig (Namespace-Sidecar überschreibt den mesh-weiten Default):

   ```yaml
   apiVersion: networking.istio.io/v1
   kind: Sidecar
   metadata:
     name: default
     namespace: frontend          # überschreibt mesh-weiten Default
   spec:
     egress:
       - hosts:
           - "./*"
           - "istio-system/*"
           - "backend/*"          # zusätzliche Abhängigkeit
   ```

2. **Discovery Selectors** nutzen, um irrelevante Namespaces (Monitoring, CI/CD, etc.) auszuschließen
3. **exportTo** auf VirtualServices/DestinationRules setzen, um Cross-Namespace-Leaks zu vermeiden
4. **istiod horizontal skalieren** ab ~200 Pods mit HPA
5. **Memory-Limits** großzügig setzen – istiod fällt bei OOM komplett aus
6. **Ambient Mode evaluieren** – besonders bei großen Clusters mit vielen Pods deutlicher Ressourcenvorteil
7. **Monitoring** der Push-Zeiten und Convergence-Time als Frühwarnsystem
8. **Canary-Updates** für istiod – Revision-basierte Upgrades (`istioctl install --revision canary`) ermöglichen parallele Control Planes

---

## 6. Hands-on Übung: Sidecar-Scope konfigurieren

### Ausgangslage

Cluster mit drei Namespaces: `frontend`, `backend`, `database`. Jeder Sidecar sieht aktuell alle Services.

### Aufgabe

1. Prüfe die aktuelle Cluster-Anzahl im Frontend-Sidecar:
   ```bash
   kubectl exec -it deploy/frontend -n frontend -c istio-proxy -- \
     pilot-agent request GET clusters | grep "^[a-z]" | wc -l
   ```

2. Erstelle eine mesh-weite Sidecar-Resource, die standardmäßig nur den eigenen Namespace erlaubt

3. Erstelle namespace-spezifische Sidecar-Resources:
   - `frontend` → sieht `backend` und `istio-system`
   - `backend` → sieht `database` und `istio-system`
   - `database` → sieht nur `istio-system`

4. Prüfe erneut die Cluster-Anzahl und vergleiche

### Erwartetes Ergebnis

Die Cluster-Anzahl pro Sidecar sollte um 50–80% sinken, je nach Mesh-Größe.
