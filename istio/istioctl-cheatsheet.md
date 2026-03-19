# istioctl Cheatsheet (Istio v1.29)

> Ohne Install-/Uninstall-/Manifest-/Profile-Kommandos. Alle Kommandos verifiziert gegen die offizielle Istio v1.29 Referenz.

---

## Version & Preflight

```bash
# Client- und Control-Plane-Version anzeigen
istioctl version
```

---

## Proxy Status (ps)

Zeigt den Sync-Status aller Envoy-Proxies mit Istiod.

```bash
# Alle Proxies im Mesh
istioctl proxy-status
istioctl ps                    # Kurzform

# Nach Namespace filtern
istioctl ps --namespace bookinfo

# Diff zwischen Envoy-Config und Istiod für einen bestimmten Proxy
istioctl ps <pod-name>.<namespace>
```

---

## Proxy Config (pc)

Envoy-Konfiguration eines Pods inspizieren.

```bash
# Listeners
istioctl proxy-config listeners <pod>.<ns>
istioctl pc l <pod>.<ns>

# Routes
istioctl pc routes <pod>.<ns>
istioctl pc r <pod>.<ns>

# Clusters (Upstream-Services)
istioctl pc clusters <pod>.<ns>
istioctl pc c <pod>.<ns>

# Endpoints
istioctl pc endpoints <pod>.<ns>
istioctl pc ep <pod>.<ns>

# Bootstrap-Konfiguration
istioctl pc bootstrap <pod>.<ns>
istioctl pc b <pod>.<ns>

# ECDS (Extension Config Discovery)
istioctl pc ecds <pod>.<ns>

# Alles auf einmal (JSON-Dump)
istioctl pc all <pod>.<ns> -o json

# Envoy Log-Level abfragen (einzelner Pod)
istioctl pc log <pod>.<ns>

# Envoy Log-Level setzen (einzelner Pod)
istioctl pc log <pod>.<ns> --level debug
istioctl pc log <pod>.<ns> --level info    # zurücksetzen

# Einzelne Envoy-Logger gezielt setzen
istioctl pc log <pod>.<ns> --level connection:debug,router:debug
istioctl pc log <pod>.<ns> --level rbac:debug,conn_handler:warning

# Per Deployment — iteriert über ALLE Pods im Deployment
istioctl pc log deploy/productpage-v1 -n bookinfo              # abfragen
istioctl pc log deploy/productpage-v1 -n bookinfo --level debug # setzen
istioctl pc log deploy/productpage-v1 -n bookinfo --level rbac:debug,conn_handler:warning

# Funktioniert analog auch mit svc/ und rs/ (ReplicaSet)
istioctl pc log svc/productpage -n bookinfo --level debug
istioctl pc log rs/productpage-v1-abc123 -n bookinfo --level debug

# Andere pc-Subcommands akzeptieren ebenfalls deploy/svc/rs
istioctl pc l deploy/productpage-v1 -n bookinfo
```

---

## Analyze

Konfiguration auf Fehler und Warnungen prüfen.

```bash
# Aktuellen Namespace analysieren
istioctl analyze

# Bestimmten Namespace
istioctl analyze -n bookinfo

# Alle Namespaces
istioctl analyze --all-namespaces

# Lokale YAML-Dateien prüfen (ohne Cluster)
istioctl analyze my-virtualservice.yaml --use-kube=false

# Bestimmte Meldungen unterdrücken
istioctl analyze -n default --suppress "IST0102=Namespace default"

# Mehrere Meldungen unterdrücken + Wildcards
istioctl analyze --all-namespaces \
  --suppress "IST0102=Namespace frod" \
  --suppress "IST0107=Pod *.baz"
```

---

## Validate

YAML-Dateien gegen das Istio-Schema validieren.

```bash
istioctl validate -f my-resource.yaml

# Kurzform
istioctl v -f my-resource.yaml

# Ganzes Verzeichnis
istioctl validate -f samples/bookinfo/networking/

# Aus stdin
kubectl get vs -o yaml | istioctl validate -f -
```

---

## Dashboard (dash / d)

Dashboards per Port-Forward öffnen.

```bash
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
istioctl dashboard zipkin

# Envoy Admin UI eines Pods
istioctl dashboard envoy <pod>.<ns>
istioctl dash envoy deploy/productpage-v1

# Proxy Dashboard (auch für ztunnel/waypoint)
istioctl dashboard proxy <pod>.<ns>

# ControlZ UI (Istiod)
istioctl dashboard controlz deploy/istiod.istio-system
```

---

## Kube-Inject

Sidecar manuell in ein Deployment injizieren.

```bash
# On-the-fly beim Apply
kubectl apply -f <(istioctl kube-inject -f deployment.yaml)

# In eine Datei schreiben
istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml

# Bestimmte Revision verwenden
istioctl kube-inject -f deployment.yaml --revision canary
```

---

## Admin Log

Istiod-Logging-Level abrufen und ändern.

```bash
# Aktuelle Log-Level von Istiod
istioctl admin log

# Bestimmten Istiod-Pod abfragen
istioctl admin log <istiod-pod>

# Log-Level ändern
istioctl admin log --level ads:debug,authorization:debug

# Alle zurücksetzen
istioctl admin log --log-reset
```

---

## Bug Report

Diagnose-Bundle für Support erstellen.

```bash
# Vollständiger Bug-Report
istioctl bug-report

# Auf bestimmte Namespaces beschränken
istioctl bug-report --include default,bookinfo

# Zeitraum begrenzen
istioctl bug-report --duration 30m
```

---

## Tag (Revision Tags)

Revision-Tags für Canary-Upgrades verwalten.

```bash
# Alle Tags auflisten
istioctl tag list

# Tag erstellen/setzen
istioctl tag set prod --revision 1-22-0

# Tag entfernen
istioctl tag remove prod
```

---

## Waypoint (Ambient Mode)

Waypoint-Proxies für Ambient Mode verwalten.

```bash
# Waypoint deployen
istioctl waypoint apply -n default
istioctl waypoint apply -n default --name my-waypoint

# Für Workloads statt Services
istioctl waypoint apply -n default --name wp --for workload

# YAML generieren (ohne Apply)
istioctl waypoint generate -n default --for service

# Alle Waypoints auflisten
istioctl waypoint list -n default
istioctl waypoint list -A                # clusterweilt

# Status prüfen
istioctl waypoint status -n default

# Waypoint löschen
istioctl waypoint delete my-waypoint -n default
istioctl waypoint delete --all -n default
```

---

## Ztunnel Config (Ambient Mode)

Ztunnel-Konfiguration inspizieren.

```bash
# Workloads
istioctl ztunnel-config workload
istioctl ztunnel-config workload <ztunnel-pod>.<ns> --node <node-name>

# Services
istioctl ztunnel-config service

# Zertifikate
istioctl ztunnel-config certificates --node <node-name>

# Policies
istioctl ztunnel-config policies

# Logging
istioctl ztunnel-config log <ztunnel-pod>.<ns>
istioctl ztunnel-config log <ztunnel-pod>.<ns> --level debug

# Alles (JSON-Dump)
istioctl ztunnel-config all <ztunnel-pod>.<ns> -o json
```

---

## Experimental (x) Kommandos

```bash
# AuthorizationPolicy eines Pods prüfen
istioctl x authz check <pod>.<ns>

# Pod beschreiben (mTLS-Status, Policies, Traffic)
istioctl x describe pod <pod> -n <ns>

# Sidecar-Injection-Status prüfen
istioctl x check-inject <pod>.<ns>
istioctl x check-inject deploy/<name> -n <ns>

# Envoy-Stats abrufen
istioctl x envoy-stats <pod>.<ns>
istioctl x envoy-stats deploy/<name> --type clusters

# Service-Metriken (benötigt Prometheus)
istioctl x metrics productpage-v1.default

# Root-CA vergleichen (Multi-Cluster)
istioctl x rootca-compare <pod1>.<ns1> <pod2>.<ns2>

# Internal Debug (Istiod xDS)
istioctl x internal-debug syncz

# VM-Workload konfigurieren
istioctl x workload entry configure -f workloadgroup.yaml -o config
istioctl x workload group create --name foo --namespace bar
```

---

## Globale Flags

| Flag | Beschreibung |
|------|-------------|
| `-n, --namespace` | Kubernetes Namespace |
| `-c, --kubeconfig` | Kubeconfig-Datei |
| `--context` | Kubernetes Context |
| `--istioNamespace` | Istio Control Plane Namespace (default: `istio-system`) |
| `--revision` | Istio Revision auswählen |
| `-o, --output` | Ausgabeformat: `json`, `yaml`, `short` |

---

## Aliase & Kurzformen

| Langform | Kurzform |
|----------|----------|
| `proxy-status` | `ps` |
| `proxy-config` | `pc` |
| `proxy-config listeners` | `pc l` |
| `proxy-config routes` | `pc r` |
| `proxy-config clusters` | `pc c` |
| `proxy-config endpoints` | `pc ep` |
| `proxy-config bootstrap` | `pc b` |
| `dashboard` | `dash` / `d` |
| `experimental` | `x` |
| `validate` | `v` |

---

## Debugging-Workflow (Kurzreferenz)

```
1. istioctl ps                              # Proxies synced?
2. istioctl analyze -n <ns>                 # Config-Fehler?
3. istioctl x describe pod <pod> -n <ns>    # Pod-Details
4. istioctl pc l <pod>.<ns> --port <port>   # Listener OK?
5. istioctl pc r <pod>.<ns> --name <port>   # Routes OK?
6. istioctl pc c <pod>.<ns> --fqdn <svc>    # Cluster/Upstream OK?
7. istioctl pc ep <pod>.<ns> --cluster <c>  # Endpoints OK?
```

---

*Quelle: [istio.io/latest/docs/reference/commands/istioctl](https://istio.io/latest/docs/reference/commands/istioctl/) — Stand: Istio v1.29*
