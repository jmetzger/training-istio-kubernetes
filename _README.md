# Istio & Kubernetes 


## Agenda
  1. Grundlagen ServiceMesh & Istio
     * [Einführung in Istio & Service Mesh-Architekturen](#einführung-in-istio--service-mesh-architekturen)
     * [Warum ein Service Mesh?](#warum-ein-service-mesh)
     * [Herausforderung & Vorteile](#herausforderung--vorteile)
     * [Architektur & Komponenten von Istio](#architektur--komponenten-von-istio)
     * [Istio Ingress Gateway vs. Kubernetes Gateway API](#istio-ingress-gateway-vs-kubernetes-gateway-api)
     * [Vergleich mit Linkerd, Cilium, Consul](#vergleich-mit-linkerd-cilium-consul)

  1. istioctl
     * [istioctl cheatsheet zum Debuggen](#istioctl-cheatsheet-zum-debuggen)

  1. Setup Cluster
     * [Self-Service Cluster ausrollen](#self-service-cluster-ausrollen)
     * [Self-Service Cluster destroy](#self-service-cluster-destroy)

  1. istio sidecar (Grundlagen)
     * [Systemanforderungen & Kubernetes-Cluster-Vorbereitung](#systemanforderungen--kubernetes-cluster-vorbereitung)
     * [Istio Proxy-Konzepte](#istio-proxy-konzepte)

  1. Installation/Konfiguration/Deinstallation von istio (sidecar - Installation mit istioctl)   
     * [Istio-Installation mit istioctl und der IstioOperator - Resource](#istio-installation-mit-istioctl-und-der-istiooperator---resource)
     * [Installations-Config-Profile](#installations-config-profile)
     * [Wie ändere ich die Config/Installation von istio - Beispiel egressGateway](#wie-ändere-ich-die-configinstallation-von-istio---beispiel-egressgateway)
     * [Deinstallation von Istio mit istioctl](#deinstallation-von-istio-mit-istioctl)

  1. Installation von istio (sidecar) mit helm-charts
     * [Systemanforderungen & Kubernetes-Cluster-Vorbereitung](#systemanforderungen--kubernetes-cluster-vorbereitung)
     * [Istio-Installation mit helm charts](#istio-installation-mit-helm-charts)
     * [istioctl als debugging - tool installieren](#istioctl-als-debugging---tool-installieren)
     * [Canary Update von istiod mit tags und revisions](#canary-update-von-istiod-mit-tags-und-revisions)

  1. Prinzip: Side-Car Injection
     * [Istio Sidecar-Injection](#istio-sidecar-injection)

  1. Demo-App (Gateway API) 
     * [Istio demo-app *bookinfo* installieren (Gateway API - Variante)](#istio-demo-app-bookinfo-installieren-gateway-api---variante)

  1. CNI-Plugin istio
     * [Vorteile und Installation mit helm](#vorteile-und-installation-mit-helm)

  1. Steuerung des Netzwerkverhaltens in Istio
     * [Istio Ingress Gateway vs. Gateway API](#istio-ingress-gateway-vs-gateway-api)
     * [Gateway API als Nachfolger von IngressGateway (Istio), VirtualService und DestinationRule](#gateway-api-als-nachfolger-von-ingressgateway-istio-virtualservice-und-destinationrule)
     * [Virtual Services, Destination Rules](#virtual-services-destination-rules)

  1. Übungen Steuerung des Netzwerkverhaltens (Gateway API)
     * [Übung: Header-basiertes Routing](#übung-header-basiertes-routing)
     * [Übung: Traffic-Shifting / Load-Balancing](#übung-traffic-shifting--load-balancing)
     * [Pfadbasiertes Routing](#pfadbasiertes-routing)
     * [Übung: LoadBalancing (LEAST_CONN) - Regeln](#übung-loadbalancing-least_conn---regeln)

  1. Service Resilience & Fehlertoleranz (Gateway API)
     * [Circuit Breaker](#circuit-breaker)
     * [Rate Limiting](#rate-limiting)
     * [Chaos Engineering mit Istio](https://istio.io/latest/docs/examples/microservices-istio/production-testing/)

  1. Service Resilience & Fehlertoleranz (nur VirtualService)
     * [Bemerkungen zu nur VirtualService und nicht Gateway API - httproute](#bemerkungen-zu-nur-virtualservice-und-nicht-gateway-api---httproute)
     * [Retries VirtualService only !](#retries-virtualservice-only-!)
     * [Fehlerinjektion (z.B. 500er) VirtualService only !](#fehlerinjektion-zb-500er-virtualservice-only-!)

  1. Debugging
     * [Debugging mit debug/run pod](#debugging-mit-debugrun-pod)

  1. Sichere Service-Kommunikation mit istio (Sicherheit)
     * [Sicherheit & Zero Trust mit Istio](#sicherheit--zero-trust-mit-istio)
     * [Was ist in istio deep-defense (defense in depth) ?](#was-ist-in-istio-deep-defense-defense-in-depth-)
     * [Überblick 3 Säulen -  Sichere Service-Kommunikation](#überblick-3-säulen----sichere-service-kommunikation)
     * [Übung Säule 1: mTLS prüfen und aktivieren/deaktivieren](#übung-säule-1-mtls-prüfen-und-aktivierendeaktivieren)
     * [Übung Säule 2 + 3: Authentication und Authorization mit JWT und RBAC](#übung-säule-2-+-3-authentication-und-authorization-mit-jwt-und-rbac)

  1. Sicherheit, Fehlertoleranz & Observability    
     * Istio Authorization Policies (ingress/egress Security)
     * [ZeroTrust für bookinfo Namespace bookinfo einrichten](#zerotrust-für-bookinfo-namespace-bookinfo-einrichten)

  1. Demo-App (istio api) 
     * [Istio demo-app *bookinfo* installieren (istio API - Variante)](#istio-demo-app-bookinfo-installieren-istio-api---variante)

  1. Übungen Steuerung des Netzwerkverhaltens (istio api)
     * [Übung: Header-basiertes Routing - istio api](#übung-header-basiertes-routing---istio-api)

  1. Monitoring, Logging & Observability
     * [Installation Jaeger](#installation-jaeger)
     * [Distributed Tracing mit Jaeger](#distributed-tracing-mit-jaeger)
     * Metriken & Dashboards mit Prometheus & Grafana
     * [Installation Prometheus Addon with Ingress](#installation-prometheus-addon-with-ingress)
     * [Metriken mit Prometheus auswerten](#metriken-mit-prometheus-auswerten)
     * [Installation Grafana Addon with Ingress](#installation-grafana-addon-with-ingress)
     * [Grafana Dashboards für istio](#grafana-dashboards-für-istio)
     * [Sinnvolle istio Metriken und Dashboards - Teil 1](#sinnvolle-istio-metriken-und-dashboards---teil-1)
     * [Sinnvolle istio Metriken und Dashboards - Teil 2](https://github.com/jmetzger/training-istio-kubernetes/blob/main/istio/observeability/istio-observeablity-in-prometheus-2.md)
     * [Installation Kiali // Installation](#installation-kiali--installation)
     * [Visualisierung mit Kiali](#visualisierung-mit-kiali)
     * [Analyse & Debugging von Service-Mesh-Daten](#analyse--debugging-von-service-mesh-daten)

  1. Skalierung, Erweiterbarkeit & Performance-Optimierung
     * [Skalierung von Istio - Möglichkeiten/Tipps&Tricks](#skalierung-von-istio---möglichkeitentippstricks)
     * [Performance-Optimierung](#performance-optimierung)
     * Sidecar-Overhead & Ressourcenoptimierung
     * [Ambient Mesh Graphical Overview (sidecar-less Istio für Performance-Gewinn)](#ambient-mesh-graphical-overview-sidecar-less-istio-für-performance-gewinn)
     * [Vergleich ohne istio, istio sidecar, istio ambient](#vergleich-ohne-istio-istio-sidecar-istio-ambient)
     * Multi-Cluster- & Hybrid-Umgebungen mit Istio
     * Istio Federation & Cross-Cluster Traffic

  1. Erweiterte Routing-Techniken & Traffic-Optimierung (Istio API) 
     * [Canary Releases & Progressive Deployments / Traffic - Shifting](#canary-releases--progressive-deployments--traffic---shifting)
     * A/B-Tests & Traffic Mirroring
     * Blue-Green- und Canary-Deployments mit Istio

  1. Installation von istio (ambient mode)
     * [istio-installation mit istioctl - ambient](#istio-installation-mit-istioctl---ambient)
     * [istio-installation with helm - ambient](#istio-installation-with-helm---ambient)
     * [Bookinfo demo in ambient ausrollen](#bookinfo-demo-in-ambient-ausrollen)

  1. Erweiterbarkeit von Istio
     * WebAssembly (Wasm) für Istio-Erweiterungen
     * [Wo läuft WASM (WebAssembly) im Rahmen von istio ?](#wo-läuft-wasm-webassembly-im-rahmen-von-istio-)
     * [Eigene Istio-Erweiterungen mit WebAssembly schreiben (RateLimit-Plugin)](#eigene-istio-erweiterungen-mit-webassembly-schreiben-ratelimit-plugin)

  1. Automatisierung mit GitOps & ArgoCD
     * [Was ist ArgoCD](#was-ist-argocd)
     * [istio ambient und bookinfo mit argocd ausrollen](#istio-ambient-und-bookinfo-mit-argocd-ausrollen)

  1. Misc
     * [Grenze bei der Header-Größe Istio vs. Nginx](#grenze-bei-der-header-größe-istio-vs-nginx)
     * [Istio als API Gateway](#istio-als-api-gateway)
     * [Mehrere Namepaces bei allowedRoutes bei Gateway - Objekt / Gateway API](#mehrere-namepaces-bei-allowedroutes-bei-gateway---objekt--gateway-api)
     * [Global Ratelimiting, je nach Kunde](#global-ratelimiting-je-nach-kunde)
     * [zu istio beitragen](#zu-istio-beitragen)

  


<div class="page-break"></div>

## Grundlagen ServiceMesh & Istio

### Einführung in Istio & Service Mesh-Architekturen


**Was ist ein Service Mesh?**
- Dedizierte Infrastrukturschicht für Service-zu-Service-Kommunikation
- Transparente Zwischenschicht ohne Code-Änderungen
- Zentrale Steuerung von Retry, Timeout, Verschlüsselung, Monitoring

**Istio im Überblick:**
- Open-Source Service Mesh (Google, IBM, Lyft)
- Nutzt Envoy-Proxies als Sidecars
- Fängt gesamten Netzwerkverkehr ab

### Vorher: Ohne Service-Mesh 

<img width="1184" height="305" alt="image" src="https://github.com/user-attachments/assets/e4e446be-b25b-465a-9682-4fc6a83c7969" />

### Nachher: Mit Service-Mesh 

<img width="763" height="895" alt="image" src="https://github.com/user-attachments/assets/18764919-344f-4696-b6c7-ced3f459dd80" />


### Mermaid-Quelltexte 

```
graph TB
    subgraph "Ohne Service Mesh"
    direction LR
    A1[Service A] -.direkter Traffic.-> B1[Service B]
    A1 -.-> C1[Service C]
    B1 -.-> C1
    end
```
```
graph TB
    subgraph "Mit Service Mesh - Sidecar Pattern"
    direction TB
    
    subgraph Pod1["Pod A"]
    direction LR
    SA[Service A] --> EA[EnvoySidecar]
    end
    
    subgraph Pod2["Pod B"]
    direction LR
    SB[Service B] --> EB[EnvoySidecar]
    end
    
    subgraph Pod3["Pod C"]
    direction LR
    SC[Service C] --> EC[EnvoySidecar]
    end

    EA -->|mTLS| EB
    EA -->|mTLS| EC
    EB -->|mTLS| EC
    end
    
    style EA fill:#4285f4
    style EB fill:#4285f4
    style EC fill:#4285f4
    style Pod1 fill:#e8f4f8
    style Pod2 fill:#e8f4f8
    style Pod3 fill:#e8f4f8
```




### Warum ein Service Mesh?


**Probleme in Microservices:**
- Dutzende/Hunderte Services → komplexe Kommunikation
- Jeder Service muss selbst implementieren:
  - Circuit Breaking
  - Load Balancing
  - mTLS-Verschlüsselung
  - Distributed Tracing
  - Retry-Logik
- Inkonsistente Implementierung über Sprachen hinweg
- Hoher Wartungsaufwand

**Lösung:**
- Komplexität aus Anwendungscode → Infrastruktur
- Platform-Teams: zentrale Policies
- Entwickler: Fokus auf Business-Logik

### Vorher: Ohne ServiceMesh 

<img width="706" height="641" alt="image" src="https://github.com/user-attachments/assets/674b1787-4903-48c7-8b1c-a72b0c424556" />


### Nachher: Mit ServiceMesh

<img width="868" height="452" alt="image" src="https://github.com/user-attachments/assets/a7bb0460-8b1e-42d9-bab6-daa7ac7a4e49" />



```
graph TD
    subgraph "Problem: Jeder Service implementiert selbst"
    SJ[Service Java] --> |implementiert| LJ[Load Balancing<br/>Retry<br/>mTLS<br/>Tracing]
    SG[Service Go] --> |implementiert| LG[Load Balancing<br/>Retry<br/>mTLS<br/>Tracing]
    SP[Service Python] --> |implementiert| LP[Load Balancing<br/>Retry<br/>mTLS<br/>Tracing]
    end
    
    subgraph "Lösung: Service Mesh übernimmt"
    S1[Service Java] --> SM[Service Mesh]
    S2[Service Go] --> SM
    S3[Service Python] --> SM
    SM --> |zentral| F[Load Balancing<br/>Retry<br/>mTLS<br/>Tracing<br/>Security<br/>Observability]
    end
    
    style LJ fill:#ff6b6b
    style LG fill:#ff6b6b
    style LP fill:#ff6b6b
    style SM fill:#51cf66
    style F fill:#51cf66
```

### Herausforderung & Vorteile


**✅ Vorteile:**
- Automatische mTLS zwischen allen Services
- Traffic-Steuerung: Canary, Blue-Green, A/B-Testing
- Einheitliches Observability (Metrics, Traces, Logs)
- Zentrale Security-Policies
- Keine Code-Änderungen nötig

**⚠️ Herausforderungen:**
- Ressourcen-Overhead: CPU/RAM pro Sidecar
- Zusätzliche Latenz (Proxy-Hops)
- Steile Lernkurve
- Komplexeres Debugging

**Wann lohnt es sich?**
- Ab ~20-30 Services
- Hohe Security/Compliance-Anforderungen
- Multi-Team-Umgebungen

<img width="1363" height="327" alt="image" src="https://github.com/user-attachments/assets/432a1941-3eed-4721-a933-b57a8243b634" />



```
graph LR
    START{Service Mesh einsetzen?}
    
    START --> Q1{Wie viele Services?}
    Q1 -->|< 10| NEIN[❌ Overhead zu hoch]
    Q1 -->|10-20| Q2{Security wichtig?}
    Q1 -->|> 20| JA[✅ Empfohlen]
    
    Q2 -->|Ja| JA
    Q2 -->|Nein| MAYBE[⚠️ Abwägen]
    
    JA --> CHECK{Ressourcen verfügbar?}
    CHECK -->|Ja| GO[✅ Service Mesh nutzen]
    CHECK -->|Nein| PLAN[📋 Ressourcen planen]
    
    style NEIN fill:#ff6b6b
    style JA fill:#51cf66
    style GO fill:#51cf66
    style MAYBE fill:#ffd93d
```

### Architektur & Komponenten von Istio


![Archiktur Istio](https://istio.io/latest/docs/ops/deployment/architecture/arch.svg)

**Data Plane:**
- Envoy-Proxies als Sidecars
- Fangen Traffic ab
- Setzen Policies durch

**Control Plane (istiod):**
- Konfigurationsverteilung
- Service Discovery
- Certificate Management
- Telemetrie-Sammlung

**Zusammenspiel:**
1. High-level Config (VirtualService, DestinationRule)
2. istiod übersetzt → Envoy-Config
3. Push an alle Proxies
4. Proxies setzen um

### Grafik (Komponenten) 

<img width="1611" height="705" alt="image" src="https://github.com/user-attachments/assets/40e5ae24-11b1-4450-9e7d-335340624d1a" />

### Grafik (Ablauf) 

<img width="1270" height="842" alt="image" src="https://github.com/user-attachments/assets/de4052d0-0140-4148-89b8-19638a25b2ae" />



**Komponenten**

```
graph TB
    subgraph "Control Plane"
    ISTIOD[istiod]
    ISTIOD --> |1. Service Discovery| SD[Service Registry]
    ISTIOD --> |2. Config Management| CM[VirtualService<br/>DestinationRule<br/>Gateway]
    ISTIOD --> |3. Certificate Authority| CA[mTLS Certificates]
    ISTIOD --> |4. Telemetry| TEL[Metrics Collection]
    end
    
    subgraph "Data Plane - Pod 1"
    E1[Envoy Proxy] --> S1[Service A]
    end
    
    subgraph "Data Plane - Pod 2"
    E2[Envoy Proxy] --> S2[Service B]
    end
    
    subgraph "Data Plane - Pod 3"
    E3[Envoy Proxy] --> S3[Service C]
    end
    
    ISTIOD -->|Config Push| E1
    ISTIOD -->|Config Push| E2
    ISTIOD -->|Config Push| E3
    
    E1 <-->|mTLS| E2
    E2 <-->|mTLS| E3
    E1 <-->|mTLS| E3
    
    E1 -.->|Telemetry| ISTIOD
    E2 -.->|Telemetry| ISTIOD
    E3 -.->|Telemetry| ISTIOD
    
    style ISTIOD fill:#4285f4
    style E1 fill:#34a853
    style E2 fill:#34a853
    style E3 fill:#34a853
```

**Traffic Flow:**
```
sequenceDiagram
    participant Admin as Admin/DevOps
    participant Istiod as Control Plane (istiod)
    participant E1 as Envoy Proxy (Service A)
    participant E2 as Envoy Proxy (Service B)
    participant S1 as Service A
    participant S2 as Service B
    
    Admin->>Istiod: kubectl apply VirtualService
    Istiod->>Istiod: Validiert Config
    Istiod->>E1: Push Envoy Config
    Istiod->>E2: Push Envoy Config
    
    S1->>E1: Outbound Request zu Service B
    E1->>E1: Load Balancing, Retry Logic
    E1->>E2: mTLS verschlüsselter Traffic
    E2->>E2: Policy Check, Rate Limiting
    E2->>S2: Weiterleitung
    S2->>E2: Response
    E2->>E1: Response
    E1->>S1: Response
    
    E1-->>Istiod: Metrics & Traces
    E2-->>Istiod: Metrics & Traces
```

### Istio Ingress Gateway vs. Kubernetes Gateway API


### Istio Ingress-Gateway-Flow (bookinfo/productpage) 

![Flow](/images/screenshot_20260314_142715.png)

### Gateway API -> Gateway Flow (bookinfo/productpage)

![Flow](/images/istio/gateway-api-flow.png)

 * Achtung:  VirtualService und DestinationRule werden teilweise immer noch benötigt, auch wenn man mit der Gateway API arbeitet,
     um bestimmte Feature abzubilden  

### Bild-Quelltext (Istio)

```
graph LR
    Client[Client/Browser]
    IG[Istio IngressGateway<br/>istio-ingressgateway Pod]
    GW[Gateway Resource<br/>istio.networking.v1beta1]
    VS[VirtualService<br/>routing rules]
    SvcA[Service A]
    SvcB[Service B]
    
    Client -->|HTTP/HTTPS| IG
    IG -.->|references| GW
    GW -.->|bound to| VS
    VS -->|route to| SvcA
    VS -->|route to| SvcB
    
    style IG fill:#e1f5ff
    style GW fill:#fff3cd
    style VS fill:#fff3cd
```

### Bild-Quelltext (Kubernetes Gateway API)

```
graph TB
    Client[Client/Browser]
    GWC[GatewayClass<br/>infrastructure.cluster.x-k8s.io]
    GW[Gateway<br/>gateway.networking.k8s.io]
    HR[HTTPRoute<br/>routing rules]
    SvcA[Service A]
    SvcB[Service B]
    
    Client -->|HTTP/HTTPS| GW
    GW -.->|instance of| GWC
    HR -.->|attached to| GW
    HR -->|route to| SvcA
    HR -->|route to| SvcB
    
    style GWC fill:#d4edda
    style GW fill:#d4edda
    style HR fill:#d4edda
```

### Vergleich mit Linkerd, Cilium, Consul


| Feature | Istio | Linkerd | Cilium | Consul |
|---------|-------|---------|--------|--------|
| **Proxy** | Envoy (C++) | Rust-Proxy | eBPF (Kernel) | Envoy |
| **Komplexität** | Hoch | Niedrig | Mittel | Mittel |
| **Overhead** | Hoch | Niedrig | Sehr niedrig | Mittel |
| **Features** | Maximal | Basis | Netzwerk-fokus | Multi-Platform |
| **K8s-Native** | Ja | Ja | Ja | Teilweise |
| **Use Case** | Enterprise, viele Features | Einfachheit | Performance | VM + K8s |

**Kernunterschiede:**
- **Linkerd:** Einfach, schnell, weniger Features
- **Cilium:** eBPF = keine Sidecars, extrem performant
- **Consul:** Multi-Plattform (VMs, Bare Metal)
- **Istio:** Feature-Champion, größte Community



## istioctl

### istioctl cheatsheet zum Debuggen


> Ohne Install-/Uninstall-/Manifest-/Profile-Kommandos. Alle Kommandos verifiziert gegen die offizielle Istio v1.29 Referenz.

---

### Version & Preflight

```bash
## Client- und Control-Plane-Version anzeigen
istioctl version
```

---

### Proxy Status (ps)

Zeigt den Sync-Status aller Envoy-Proxies mit Istiod.

```bash
## Alle Proxies im Mesh
istioctl proxy-status
istioctl ps                    # Kurzform

## Nach Namespace filtern
istioctl ps --namespace bookinfo

## Diff zwischen Envoy-Config und Istiod für einen bestimmten Proxy
istioctl ps <pod-name>.<namespace>
```

---

### Proxy Config (pc)

Envoy-Konfiguration eines Pods inspizieren.

```bash
## Listeners
istioctl proxy-config listeners <pod>.<ns>
istioctl pc l <pod>.<ns>

## Routes
istioctl pc routes <pod>.<ns>
istioctl pc r <pod>.<ns>

## Clusters (Upstream-Services)
istioctl pc clusters <pod>.<ns>
istioctl pc c <pod>.<ns>

## Endpoints
istioctl pc endpoints <pod>.<ns>
istioctl pc ep <pod>.<ns>

## Bootstrap-Konfiguration
istioctl pc bootstrap <pod>.<ns>
istioctl pc b <pod>.<ns>

## ECDS (Extension Config Discovery)
istioctl pc ecds <pod>.<ns>

## Alles auf einmal (JSON-Dump)
istioctl pc all <pod>.<ns> -o json

## Envoy Log-Level abfragen (einzelner Pod)
istioctl pc log <pod>.<ns>

## Envoy Log-Level setzen (einzelner Pod)
istioctl pc log <pod>.<ns> --level debug
istioctl pc log <pod>.<ns> --level info    # zurücksetzen

## Einzelne Envoy-Logger gezielt setzen
istioctl pc log <pod>.<ns> --level connection:debug,router:debug
istioctl pc log <pod>.<ns> --level rbac:debug,conn_handler:warning

## Per Deployment — iteriert über ALLE Pods im Deployment
istioctl pc log deploy/productpage-v1 -n bookinfo              # abfragen
istioctl pc log deploy/productpage-v1 -n bookinfo --level debug # setzen
istioctl pc log deploy/productpage-v1 -n bookinfo --level rbac:debug,conn_handler:warning

## Funktioniert analog auch mit svc/ und rs/ (ReplicaSet)
istioctl pc log svc/productpage -n bookinfo --level debug
istioctl pc log rs/productpage-v1-abc123 -n bookinfo --level debug

## Andere pc-Subcommands akzeptieren ebenfalls deploy/svc/rs
istioctl pc l deploy/productpage-v1 -n bookinfo
```

---

### Analyze

Konfiguration auf Fehler und Warnungen prüfen.

```bash
## Aktuellen Namespace analysieren
istioctl analyze

## Bestimmten Namespace
istioctl analyze -n bookinfo

## Alle Namespaces
istioctl analyze --all-namespaces

## Lokale YAML-Dateien prüfen (ohne Cluster)
istioctl analyze my-virtualservice.yaml --use-kube=false

## Bestimmte Meldungen unterdrücken
istioctl analyze -n default --suppress "IST0102=Namespace default"

## Mehrere Meldungen unterdrücken + Wildcards
istioctl analyze --all-namespaces \
  --suppress "IST0102=Namespace frod" \
  --suppress "IST0107=Pod *.baz"
```

---

### Validate

YAML-Dateien gegen das Istio-Schema validieren.

```bash
istioctl validate -f my-resource.yaml

## Kurzform
istioctl v -f my-resource.yaml

## Ganzes Verzeichnis
istioctl validate -f samples/bookinfo/networking/

## Aus stdin
kubectl get vs -o yaml | istioctl validate -f -
```

---

### Dashboard (dash / d)

Dashboards per Port-Forward öffnen.

```bash
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
istioctl dashboard zipkin

## Envoy Admin UI eines Pods
istioctl dashboard envoy <pod>.<ns>
istioctl dash envoy deploy/productpage-v1

## Proxy Dashboard (auch für ztunnel/waypoint)
istioctl dashboard proxy <pod>.<ns>

## ControlZ UI (Istiod)
istioctl dashboard controlz deploy/istiod.istio-system
```

---

### Kube-Inject

Sidecar manuell in ein Deployment injizieren.

```bash
## On-the-fly beim Apply
kubectl apply -f <(istioctl kube-inject -f deployment.yaml)

## In eine Datei schreiben
istioctl kube-inject -f deployment.yaml -o deployment-injected.yaml

## Bestimmte Revision verwenden
istioctl kube-inject -f deployment.yaml --revision canary
```

---

### Admin Log

Istiod-Logging-Level abrufen und ändern.

```bash
## Aktuelle Log-Level von Istiod
istioctl admin log

## Bestimmten Istiod-Pod abfragen
istioctl admin log <istiod-pod>

## Log-Level ändern
istioctl admin log --level ads:debug,authorization:debug

## Alle zurücksetzen
istioctl admin log --log-reset
```

---

### Bug Report

Diagnose-Bundle für Support erstellen.

```bash
## Vollständiger Bug-Report
istioctl bug-report

## Auf bestimmte Namespaces beschränken
istioctl bug-report --include default,bookinfo

## Zeitraum begrenzen
istioctl bug-report --duration 30m
```

---

### Tag (Revision Tags)

Revision-Tags für Canary-Upgrades verwalten.

```bash
## Alle Tags auflisten
istioctl tag list

## Tag erstellen/setzen
istioctl tag set prod --revision 1-22-0

## Tag entfernen
istioctl tag remove prod
```

---

### Waypoint (Ambient Mode)

Waypoint-Proxies für Ambient Mode verwalten.

```bash
## Waypoint deployen
istioctl waypoint apply -n default
istioctl waypoint apply -n default --name my-waypoint

## Für Workloads statt Services
istioctl waypoint apply -n default --name wp --for workload

## YAML generieren (ohne Apply)
istioctl waypoint generate -n default --for service

## Alle Waypoints auflisten
istioctl waypoint list -n default
istioctl waypoint list -A                # clusterweilt

## Status prüfen
istioctl waypoint status -n default

## Waypoint löschen
istioctl waypoint delete my-waypoint -n default
istioctl waypoint delete --all -n default
```

---

### Ztunnel Config (Ambient Mode)

Ztunnel-Konfiguration inspizieren.

```bash
## Workloads
istioctl ztunnel-config workload
istioctl ztunnel-config workload <ztunnel-pod>.<ns> --node <node-name>

## Services
istioctl ztunnel-config service

## Zertifikate
istioctl ztunnel-config certificates --node <node-name>

## Policies
istioctl ztunnel-config policies

## Logging
istioctl ztunnel-config log <ztunnel-pod>.<ns>
istioctl ztunnel-config log <ztunnel-pod>.<ns> --level debug

## Alles (JSON-Dump)
istioctl ztunnel-config all <ztunnel-pod>.<ns> -o json
```

---

### Experimental (x) Kommandos

```bash
## AuthorizationPolicy eines Pods prüfen
istioctl x authz check <pod>.<ns>

## Pod beschreiben (mTLS-Status, Policies, Traffic)
istioctl x describe pod <pod> -n <ns>

## Sidecar-Injection-Status prüfen
istioctl x check-inject <pod>.<ns>
istioctl x check-inject deploy/<name> -n <ns>

## Envoy-Stats abrufen
istioctl x envoy-stats <pod>.<ns>
istioctl x envoy-stats deploy/<name> --type clusters

## Service-Metriken (benötigt Prometheus)
istioctl x metrics productpage-v1.default

## Root-CA vergleichen (Multi-Cluster)
istioctl x rootca-compare <pod1>.<ns1> <pod2>.<ns2>

## Internal Debug (Istiod xDS)
istioctl x internal-debug syncz

## VM-Workload konfigurieren
istioctl x workload entry configure -f workloadgroup.yaml -o config
istioctl x workload group create --name foo --namespace bar
```

---

### Globale Flags

| Flag | Beschreibung |
|------|-------------|
| `-n, --namespace` | Kubernetes Namespace |
| `-c, --kubeconfig` | Kubeconfig-Datei |
| `--context` | Kubernetes Context |
| `--istioNamespace` | Istio Control Plane Namespace (default: `istio-system`) |
| `--revision` | Istio Revision auswählen |
| `-o, --output` | Ausgabeformat: `json`, `yaml`, `short` |

---

### Aliase & Kurzformen

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

### Debugging-Workflow (Kurzreferenz)

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

## Setup Cluster

### Self-Service Cluster ausrollen


  * ausgerollt mit terraform (binary ist installiert) - snap install --classic terraform 
  * beinhaltet
      1. 1 controlplane
      1. 3 worker nodes
      1. metallb mit ip's (IP-Adressen) der Nodes (hacky but works)
      1. ingress mit wildcard-domain:  *.tlnx.do.t3isp.de

### Vorbereitung seitens des Trainers

```
## /tmp/.env - Datei wurde vom Trainer vorbereitet
## Inhalt / export -> damit Umgebungsvariable 
export TF_VAR_do_token="DAS_TOKEN_FUER_DIGITALOCEAN"
```

```
Folgende Berechtigungen wurden für das Token gesetzt
```

<img width="1536" height="595" alt="image" src="https://github.com/user-attachments/assets/b394d2b6-82c7-4be6-b8d6-51afc9fd1944" />
   
### Walktrough 

  * Setup takes about 6-7 minutes
  * Hinweis: /tmp/.env beinhaltet Digitalocean Access Token der für das einrichten benötigt wird.

```
cd
git clone https://github.com/jmetzger/training-istio-kubernetes-stack-do-terraform.git install
cd install
cat /tmp/.env
source /tmp/.env
tofu init
tofu apply -auto-approve
```
### Hinweis

```
## Sollte es nicht sauber durchlaufen
## Cluster nochmal löschen 
./scripts/safe-destroy.sh
tofu apply -auto-approve
```


### Self-Service Cluster destroy


```
cd 
cd install 
source /tmp/.env 
terraform destroy -auto-approve
```

## istio sidecar (Grundlagen)

### Systemanforderungen & Kubernetes-Cluster-Vorbereitung


### Hardware-Anforderungen

#### Master Node (Control Plane)
- **Minimum:** 2 CPU, 2 GB RAM, 20 GB Disk
- **Empfohlen:** 4 CPU, 8 GB RAM, 50 GB Disk
- **Schulung:** 2 CPU, 4 GB RAM, 30 GB Disk

#### Worker Node
- **Minimum:** 2 CPU, 2 GB RAM, 20 GB Disk
- **Empfohlen:** 4 CPU, 16 GB RAM, 100 GB Disk
- **Schulung:** 2 CPU, 4 GB RAM, 30 GB Disk

### Software-Voraussetzungen

#### Betriebssystem
- Ubuntu 20.04/22.04 LTS
- Debian 11/12
- RHEL/Rocky/Alma Linux 8/9
- 64-bit Architektur erforderlich

#### Container Runtime
- containerd (empfohlen)
- CRI-O
- Docker Engine (über cri-dockerd)

#### System-Tools
```bash
curl, wget, apt-transport-https, ca-certificates
```

### Netzwerk-Anforderungen

#### Ports Control Plane
- 6443: Kubernetes API Server
- 2379-2380: etcd
- 10250: Kubelet API
- 10259: kube-scheduler
- 10257: kube-controller-manager

#### Ports Worker Nodes
- 10250: Kubelet API
- 30000-32767: NodePort Services

### Cluster-Vorbereitung

#### System-Updates
```bash
apt update && apt upgrade -y
## oder
dnf update -y
```

#### Swap deaktivieren
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

#### Kernel-Module laden
```bash
cat <<EOL | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOL

modprobe overlay
modprobe br_netfilter
```

#### Sysctl-Parameter
```bash
cat <<EOL | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOL

sysctl --system
```

#### Firewall
- UFW/firewalld deaktivieren (Schulung)
- Oder: Benötigte Ports freischalten



lsmod | grep br_netfilter
```

### Istio Proxy-Konzepte


### 1. Was ist Envoy?

Envoy ist ein High-Performance L4/L7 Proxy, entwickelt von Lyft und heute ein CNCF-Graduated-Projekt. Er ist in C++ geschrieben und wurde von Anfang an für dynamische, Cloud-native Umgebungen konzipiert.

Istio nutzt Envoy als **Data Plane** — jeder Proxy im Mesh ist eine Envoy-Instanz. Istio erweitert Envoy über eigene Filter (z.B. für mTLS, Telemetrie) und steuert ihn zentral über istiod (Control Plane).

### 2. Wo sitzt der Proxy?

Der Envoy Proxy läuft als **Native Sidecar** im gleichen Pod wie die Applikation.

Ab Kubernetes 1.28+ wird `istio-proxy` als Init-Container mit `restartPolicy: Always` definiert (Native Sidecar Feature). Das bedeutet:

- Er erscheint unter `spec.initContainers`, läuft aber für die gesamte Pod-Laufzeit
- Er startet **vor** der App und stoppt **nach** der App
- `kubectl get pods` zeigt ihn trotzdem als laufenden Container (z.B. `2/2 Ready`)

Da er sich das **Network Namespace** mit dem App-Container teilt, sieht er sämtlichen ein- und ausgehenden Traffic des Pods.

```
Pod
├── initContainers
│   ├── istio-init        ← setzt iptables-Regeln (entfällt mit CNI Plugin)
│   └── istio-proxy       ← Native Sidecar (Envoy), läuft dauerhaft
└── containers
    └── app               ← Applikations-Container
```

### 3. Wie kommt der Traffic zum Proxy?

Der Traffic wird **transparent** zum Envoy umgeleitet — die Applikation bemerkt nichts davon. Dafür werden iptables-Regeln gesetzt, die allen ein- und ausgehenden Traffic über den Envoy-Proxy routen.

#### Variante A: `istio-init` Init-Container (Standard)

- Läuft einmalig vor dem App-Start
- Setzt iptables-Regeln im Network Namespace des Pods
- Benötigt `NET_ADMIN` und `NET_RAW` Capabilities

#### Variante B: Istio CNI Plugin (empfohlen)

- Arbeitet als Chained CNI Plugin auf Node-Ebene
- Setzt die gleichen iptables-Regeln, aber **bevor** irgendein Container startet
- `istio-init` Init-Container entfällt komplett

**Vorteile CNI Plugin:**

- Keine `NET_ADMIN`/`NET_RAW` Capabilities im Pod → kompatibel mit `restricted` Pod Security Standard
- Keine Race Conditions → kein Traffic geht am Proxy vorbei, auch wenn andere Init-Container Netzwerk-Requests machen
- Sicherheitsaspekt: Ohne CNI Plugin können Init-Container, die vor `istio-init` laufen, unverschlüsselt und ohne Policy kommunizieren (kein mTLS, keine AuthorizationPolicy)
- Im Ambient Mode zwingend erforderlich

**Hinweis:** Das CNI Plugin setzt die identischen iptables-Regeln wie `istio-init` — es gibt keinen Runtime-Performance-Unterschied.

### 4. Was macht der Proxy?

Envoy übernimmt als Sidecar eine Vielzahl von Aufgaben, ohne dass die Applikation angepasst werden muss:

| Bereich | Funktion |
|---------|----------|
| **Security** | mTLS (automatische Verschlüsselung zwischen Services), Zertifikatsrotation |
| **Traffic Management** | Routing, Retries, Timeouts, Fault Injection, Traffic Shifting (Canary, Blue/Green) |
| **Resilienz** | Circuit Breaking, Outlier Detection, Rate Limiting |
| **Load Balancing** | Round Robin, Least Connections, Random, Consistent Hashing |
| **Observability** | Metrics (RED: Rate, Errors, Duration), Distributed Traces, Access Logs |

All das wird **nicht in der App konfiguriert**, sondern über Istio-Ressourcen (siehe nächster Abschnitt).

### 5. Wie wird der Proxy konfiguriert?

Envoy wird **nie direkt konfiguriert**. Stattdessen:

1. Der Anwender erstellt Istio CRDs (z.B. VirtualService, DestinationRule, AuthorizationPolicy)
2. **istiod** (Control Plane) übersetzt diese in Envoy-native Konfiguration
3. istiod pusht die Konfiguration über die **xDS API** (gRPC-Stream) an die Envoy-Proxies

#### xDS API — Hot Reload ohne Neustart

Envoy öffnet eine langlebige gRPC-Verbindung zu istiod (Watch/Subscribe-Pattern). Bei Änderungen pusht istiod die neue Konfiguration über den bestehenden Stream.

**Wichtig:** Envoy wendet Änderungen **in-memory** an — kein Schreiben auf die Platte, kein Prozess-Reload. Bestehende Connections werden nicht unterbrochen. Das unterscheidet Envoy fundamental von klassischen Proxies wie Nginx oder HAProxy, die ein Config-File und einen Reload benötigen.

Die einzige Datei-basierte Config ist die **Bootstrap-Konfiguration** beim ersten Start. Sie enthält im Wesentlichen nur die Verbindungsdaten zu istiod (Port 15012). Alles Weitere kommt dynamisch über xDS.

### 6. Sidecar Injection

Der Envoy Sidecar wird über einen **Mutating Admission Webhook** in den Pod injiziert. Es gibt zwei Varianten:

#### Automatic Injection (empfohlen)

Namespace mit Label versehen:

```bash
kubectl label namespace <NAMESPACE> istio-injection=enabled
```

Alle neuen Pods in diesem Namespace erhalten automatisch den Sidecar.

#### Manual Injection

```bash
istioctl kube-inject -f deployment.yaml | kubectl apply -f -
```

Nützlich für einzelne Workloads oder zum Debugging (um die generierte Pod-Spec zu inspizieren).

### 7. Memory-Overhead

Envoy bekommt per Default die Konfiguration für **alle erreichbaren Services** im Mesh — nicht nur die, die der Pod tatsächlich anspricht. In großen Meshes führt das zu erheblichem Speicherverbrauch:

| Mesh-Größe | Typischer RAM pro Sidecar |
|------------|---------------------------|
| Klein (< 20 Services) | 50–80 MB |
| Mittel (50–100 Services) | 100–200 MB |
| Groß (100+ Services) | 200–300+ MB |

#### Gegenmaßnahme: Sidecar CRD

Mit der Istio Sidecar CRD kann die Sichtbarkeit eines Envoy-Proxies eingeschränkt werden:

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: frontend-sidecar
  namespace: frontend
spec:
  egress:
  - hosts:
    - "./backend.backend.svc.cluster.local"
    - "istio-system/*"
```

Der Envoy kennt dann nur noch die explizit genannten Services → deutlich weniger Memory.

**Vorsicht:** Falsch konfiguriert bricht die Kommunikation, weil der Envoy den Ziel-Service nicht kennt. Service-Abhängigkeiten müssen explizit gepflegt werden.

#### Alternative: Ambient Mode

Im Ambient Mode entfällt der Envoy-Sidecar pro Pod komplett. Stattdessen:

- **ztunnel** (pro Node) für L4 (mTLS, TCP-Routing)
- **Waypoint Proxies** (optional, pro Namespace/Service) für L7 (HTTP-Routing, Retries etc.)

Der Memory-Overhead reduziert sich erheblich, da nicht mehr jeder Pod seinen eigenen Envoy betreibt.

### 8. Jobs und CronJobs im Mesh

#### Problem (vor Native Sidecars)

Ein Job führt seinen Task aus → App-Container beendet sich → `istio-proxy` Sidecar läuft weiter → Pod bleibt `Running` statt `Completed` → Job wird nie als fertig erkannt.

Folgen:

- CronJobs häufen sich, weil der vorherige nie abschließt
- `activeDeadlineSeconds` / `backoffLimit` greifen → Job wird als **Failed** markiert
- Ressourcen bleiben durch Zombie-Envoys belegt

#### Workarounds (vor K8s 1.28)

```bash
## Envoy manuell herunterfahren am Ende des Job-Scripts
curl -X POST http://localhost:15000/quitquitquit
```

#### Lösung: Native Sidecars (K8s 1.28+)

Mit Native Sidecars weiß Kubernetes, dass `istio-proxy` ein Sidecar ist und beendet ihn automatisch, sobald alle regulären Container fertig sind. Keine Workarounds mehr nötig.

### 9. Zusammenfassung

```
┌─────────────────────────────────────────────────┐
│                    istiod                        │
│              (Control Plane)                     │
│   Übersetzt CRDs → Envoy-Config via xDS API     │
└──────────────────┬──────────────────────────────┘
                   │ gRPC (xDS Push)
                   ▼
┌─────────────────────────────────────────────────┐
│                    Pod                           │
│  ┌─────────────────────────────────────────┐    │
│  │ istio-proxy (Native Sidecar / Envoy)    │    │
│  │ • mTLS, Routing, Retries, Telemetrie    │    │
│  │ • Config in-memory, hot reload          │    │
│  └────────────────┬────────────────────────┘    │
│         iptables  │  (via istio-init oder CNI)  │
│  ┌────────────────▼────────────────────────┐    │
│  │ App-Container                           │    │
│  │ (sieht nichts vom Proxy)                │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

## Installation/Konfiguration/Deinstallation von istio (sidecar - Installation mit istioctl)   

### Istio-Installation mit istioctl und der IstioOperator - Resource


  * Most simplistic way
  * Doing the right setup is done with profiles
  * Interestingly it uses an compile-in helm chart  (see also: Show what a profile does)

### Hint for production 

  * Best option (in most cases) is default 

### in our case: Including demo (tracing is activated) 

  * Not suitable for production !!

### Show what a profile does 

```
istioctl manifest generate > istio-manifest.yaml
## If not profile is mentioned, it uses the default profile
## it does not use an operator 
cat istio-manifest.yaml | grep -i -A20 "^Kind" | less
## If you want you can apply it like so:
## kubectl apply -f istio-manifest.yaml 

```

### Installation including Demo 

> [!CAUTION]
> This profile (demo) enables high levels of tracing and access logging so it is not suitable for performance tests.

#### Schritt 1: istio runterladen und installieren 

```
cd 
## current version of istio is 1.28.0
curl -L https://istio.io/downloadIstio | sh -
ln -s ~/istio-1.28.0 ~/istio
echo "export PATH=~/istio-1.28.0/bin:$PATH" >> ~/.bashrc
source ~/.bashrc 
```

#### Schritt 2: bash completion integrieren 

```
cp ~/istio/tools/istioctl.bash ~/istioctl.bash
echo "source ~/istioctl.bash" >> ~/.bashrc
source ~/istioctl.bash
```

#### Schritt 2.5. See what it would install 

```
## dry-run 
istioctl install -f ~/istio/samples/bookinfo/demo-profile-no-gateways.yaml -y --dry-run
```

#### Schritt 3: Installation with demo (by using operator)

```
## cat ~/istio/samples/bookinfo/demo-profile-no-gateways.yaml
## Wird vom ControlPlane ausgewertet
## Hier wird das ingressgateway abgeschaltet,
## Weil wir das nicht benötigen, wenn wir
## die Kubernetes Gateway API verwenden 
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false
    egressGateways:
    - name: istio-egressgateway
      enabled: false
```


```
## Der Trend geht Richtung Kubernetees Gateway API
istioctl install -f ~/istio/samples/bookinfo/demo-profile-no-gateways.yaml -y
```

#### Schritt 4: Gateway API's CRD's installieren 

```
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.4.0" | kubectl apply -f -; }
```

### Reference: Get started 

  * https://istio.io/latest/docs/setup/getting-started/

### Installations-Config-Profile


  * istio verwendet verschiedene vorgefertigte Profile, die das Ausrollen (installieren) erleichtern
  * Diese können in istioctl verwendet werden, aber auch mit dem helm-chart

### Welcher Profile gibt es ? 

  * Es gibt deployment - profile und platform profile

### Übersicht der Deployment - Profile 

| Core components        | default | demo | minimal | remote | empty | preview | ambient |
|------------------------|---------|------|---------|--------|-------|---------|---------|
| istio-egressgateway    |         | ✓    |         |        |       |         |         |
| istio-ingressgateway   | ✓       | ✓   |         |        |       | ✓       |         |
| istiod                 | ✓       | ✓    | ✓       |        |       | ✓       | ✓       |
| CNI                    |         |      |         |        |       |         | ✓       |
| Ztunnel                |         |      |         |        |       |         | ✓       |

### Welches Deployment - Profile nehme ich am besten 

  * Für Productiion am besten Default (mit Sidecar) oder Ambient (für den Ambient - Modus)
  * Zum Testen / Üben demo (mit Sidecar), aber nicht ! für die Production. Schlecht für Performance (Hier ist ganz viel Debuggen und Tracing aktiviert)

### Übersicht Platform - Profile 

| Plattform | Beschreibung |
|-----------|--------------|
| gke | Sets chart options required or recommended for installing Istio in Google Kubernetes Engine (GKE) environments. |
| eks | Sets chart options required or recommended for installing Istio in Amazon's Elastic Kubernetes Service (EKS) environments. |
| openshift | Sets chart options required or recommended for installing Istio in OpenShift environments. |
| k3d | Sets chart options required or recommended for installing Istio in k3d environments. |
| k3s | Sets chart options required or recommended for installing Istio in K3s environments. |
| microk8s | Sets chart options required or recommended for installing Istio in MicroK8s environments. |
| minikube | Sets chart options required or recommended for installing Istio in minikube environments. |

### Reference:

  * https://istio.io/latest/docs/setup/additional-setup/config-profiles/

### Wie ändere ich die Config/Installation von istio - Beispiel egressGateway


  * Always !! mention the profile
  * Overlays work for: Properties and Scalars, but not for lists
  * To avoid problems always ! list the complete config

### Exercise: Enable egress gateway 

#### Step 1: Prepare istioOperator - config 

```
cd
mkdir -p manifests/istio 
cd manfifests/istio
nano istioOperator.yaml
```

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false       # keep this, so it does NOT come back
    egressGateways:
    - name: istio-egressgateway
      enabled: true        # turn this on
```


### Step 2: Dry-run first 

```
## See what is done 
istiocl install -f istioOperator.yaml --dry-run 
```

### Step 3: Install (Change now) 

```
istioclt install -f istioOperator.yaml 

```

### Deinstallation von Istio mit istioctl


```
istioctl uninstall -y --purge
```

## Installation von istio (sidecar) mit helm-charts

### Systemanforderungen & Kubernetes-Cluster-Vorbereitung


### Hardware-Anforderungen

#### Master Node (Control Plane)
- **Minimum:** 2 CPU, 2 GB RAM, 20 GB Disk
- **Empfohlen:** 4 CPU, 8 GB RAM, 50 GB Disk
- **Schulung:** 2 CPU, 4 GB RAM, 30 GB Disk

#### Worker Node
- **Minimum:** 2 CPU, 2 GB RAM, 20 GB Disk
- **Empfohlen:** 4 CPU, 16 GB RAM, 100 GB Disk
- **Schulung:** 2 CPU, 4 GB RAM, 30 GB Disk

### Software-Voraussetzungen

#### Betriebssystem
- Ubuntu 20.04/22.04 LTS
- Debian 11/12
- RHEL/Rocky/Alma Linux 8/9
- 64-bit Architektur erforderlich

#### Container Runtime
- containerd (empfohlen)
- CRI-O
- Docker Engine (über cri-dockerd)

#### System-Tools
```bash
curl, wget, apt-transport-https, ca-certificates
```

### Netzwerk-Anforderungen

#### Ports Control Plane
- 6443: Kubernetes API Server
- 2379-2380: etcd
- 10250: Kubelet API
- 10259: kube-scheduler
- 10257: kube-controller-manager

#### Ports Worker Nodes
- 10250: Kubelet API
- 30000-32767: NodePort Services

### Cluster-Vorbereitung

#### System-Updates
```bash
apt update && apt upgrade -y
## oder
dnf update -y
```

#### Swap deaktivieren
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

#### Kernel-Module laden
```bash
cat <<EOL | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOL

modprobe overlay
modprobe br_netfilter
```

#### Sysctl-Parameter
```bash
cat <<EOL | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOL

sysctl --system
```

#### Firewall
- UFW/firewalld deaktivieren (Schulung)
- Oder: Benötigte Ports freischalten



lsmod | grep br_netfilter
```

### Istio-Installation mit helm charts


### Voraussetzungen

- Kubernetes-Cluster läuft und `kubectl` ist konfiguriert
- Helm ist installiert (`helm version`)

### Profile 

  * Die Charts von istio verwenden das "profile"-Konzept
  * Diese werden nachher einfach mit dem defaults - Block gemerged
  * Wenn wir z.B. eine Einstellungen für die Demo haben wollen (z.B. sampleTracing auf 100 statt auf 1), können wir das demo-profile verwenden
  * Die Profile für die Helm-Charts findest Du hier (nicht alles, was es für die Installation mit istioctl gibt, gibt es auch mit helm !!): 
    [Profile für Helm]https://github.com/istio/istio/tree/release-1.29.1/manifests/helm-profiles)
  * Wichtig ! Sie werden nicht in allen Charts verwenden, wenn wir es überall angeben, müssen wir uns aber keine Gedanken machen, in welchem Start 


### Schritt 1: Helm-Repository hinzufügen

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### Schritt 2: Namespace erstellen und Base-Chart installieren

Das Base-Chart installiert die Istio CRDs (Custom Resource Definitions):

```bash
helm install istio-base istio/base -n istio-system --set defaultRevision=default --set profile=demo --create-namespace --version 1.29.1
```

Prüfen:

```bash
helm ls -n istio-system
```

Der Status von `istio-base` sollte `deployed` sein.

### Schritt 3: Istiod (Control Plane) installieren

```bash
helm install istiod istio/istiod -n istio-system --wait --version 1.29.1  --set profile=demo
```

Prüfen:

```bash
helm ls -n istio-system
kubectl get deployments -n istio-system
```

`istiod` sollte `READY 1/1` anzeigen.

### Schritt 4 : Ingress Gateway installieren

```bash
helm install istio-ingress istio/gateway -n istio-ingress --wait --version 1.29.1 --set profile=demo --create-namespace
```

> **Hinweis:** Der Gateway-Namespace darf kein Label `istio-injection=disabled` haben.

### Schritt 4.5: (Optional) Metrics-Server installieren 

  * Der HorizontalPodAutoscaler der vom Gateway - Chart verwendet wird braucht den Metrics-Server um die CPU vom istio-ingressgateway - Pod auszulesen
  * kubectl -n istio-ingress get hpa # Hier steht aktuell **Unknown**

<img width="1527" height="94" alt="image" src="https://github.com/user-attachments/assets/9000a286-3407-452a-8092-43882df72817" />

  * Um das zu beheben müssen wir den Metrics-Server installieren




```
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
```


```
cd 
mkdir -p helm-charts/metrics-server
cd helm-charts/metrics-server
nano values.yaml
```

```
args:
  - --kubelet-insecure-tls
```



```
helm -n kube-system upgrade --install metrics-server metrics-server/metrics-server --version 3.13.0 --create-namespace -f values.yaml 
```

```
## Überprüfen, es dauert ein bisschen, bis er Ready
kubectl -n kube-system get pods
```

```
## hpa sollte jetzt die richtigen Infos haben
kubectl -n istio-ingress get hpa 
```

<img width="1408" height="95" alt="image" src="https://github.com/user-attachments/assets/192af4aa-a741-4d58-bc9c-c65f04b1395f" />


### Konfiguration anpassen

Default-Werte eines Charts anzeigen:

```bash
helm show values istio/istiod
helm show values istio/gateway
```

Eigene Werte beim Installieren mitgeben:

```bash
helm install istiod istio/istiod -n istio-system --values my-values.yaml --wait
```

### Deinstallation

In umgekehrter Reihenfolge deinstallieren:

```bash
## 1. Gateway (falls installiert)
helm delete istio-ingress -n istio-ingress
kubectl delete namespace istio-ingress

## 2. Istiod
helm delete istiod -n istio-system

## 3. Base-Chart
helm delete istio-base -n istio-system

## 4. Namespace aufräumen
kubectl delete namespace istio-system
```

Optional – CRDs entfernen (Achtung: löscht alle Istio-Ressourcen!):

```bash
kubectl get crd -oname | grep 'istio.io' | xargs kubectl delete
```

### Referenz

- [Offizielle Doku](https://istio.io/latest/docs/setup/install/helm/)
- [Helm Upgrade Guide](https://istio.io/latest/docs/setup/upgrade/helm/)
- [Advanced Helm Customization](https://istio.io/latest/docs/setup/additional-setup/customize-installation-helm/)

### istioctl als debugging - tool installieren


  * Good tool for debugging
  * Download includes demo and additional tools  

#### Schritt 1: istio runterladen und installieren 

```
cd 
## current version of istio is 1.29.1
curl -L https://istio.io/downloadIstio | sh -
ln -s ~/istio-1.29.1 ~/istio
echo 'export PATH=~/istio/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

#### Schritt 2: bash completion integrieren 

```
cp ~/istio/tools/istioctl.bash ~/istioctl.bash
echo "source ~/istioctl.bash" >> ~/.bashrc
source ~/istioctl.bash
```


### Canary Update von istiod mit tags und revisions


### Überblick

Revision Tags sind stabile Aliase für Istio-Revisionen. Namespaces referenzieren nur den Tag-Namen – beim Upgrade wird der Tag auf die neue Revision umgeschwenkt, ohne Namespace-Labels ändern zu müssen.

```
Namespace (istio.io/rev: prod) ──► Tag "prod" ──► istiod Revision 1-22-0
                                                    ↓  (nach Upgrade)
                                                   istiod Revision 1-23-0
```

---

### Voraussetzungen

```bash
## Helm Repo hinzufügen
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

## istioctl installieren (wird für Tag-Management benötigt)
curl -sL https://istio.io/downloadIstio | ISTIO_VERSION=1.22.0 sh -
export PATH=$PWD/istio-1.22.0/bin:$PATH
```

---

### Schritt 1: Initiale Installation (Revision 1-22-0)

#### 1.1 Istio Base (CRDs)

```bash
helm install istio-base istio/base \
  -n istio-system \
  --create-namespace \
  --version 1.22.0
```

#### 1.2 Istiod mit Revision

> **Wichtig:** Bei Revision-basierter Installation wird `--set revision=` gesetzt.
> Der Revisionsname darf nur alphanumerische Zeichen und `-` enthalten (keine Punkte).

```bash
helm install istiod-1-22-0 istio/istiod \
  -n istio-system \
  --version 1.22.0 \
  --set revision=1-22-0 \
  --wait
```

Prüfen:

```bash
kubectl get pods -n istio-system -l app=istiod
## NAME                            READY   STATUS
## istiod-1-22-0-xxxxxxxxx-xxxxx   1/1     Running
```

#### 1.3 Gateway (optional)

```bash
helm install istio-ingressgateway istio/gateway \
  -n istio-ingress \
  --create-namespace \
  --version 1.22.0 \
  --set revision=1-22-0
```

---

### Schritt 2: Revision Tag erstellen

```bash
## Tag "prod" zeigt auf Revision 1-22-0
istioctl tag set prod --revision 1-22-0

## Prüfen
istioctl tag list
## TAG    REVISION
## prod   1-22-0
```

Technisch erzeugt das eine `MutatingWebhookConfiguration` namens `istio-revision-tag-prod`.

---

### Schritt 3: Namespaces labeln

```bash
## Namespace mit dem Tag (nicht der Revision!) labeln
kubectl label namespace default istio.io/rev=prod

## Falls noch das alte (nicht-revisionierte) Label existiert, entfernen!
## "istio-injection=enabled" hat Vorrang vor "istio.io/rev" und würde
## den Revision-Tag ignorieren. Nur nötig wenn vorher ohne Revisionen installiert.
kubectl label namespace default istio-injection-
```

Demo-App deployen:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.22/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl rollout status deployment -n default
```

Sidecar-Injection prüfen:

```bash
kubectl get pods -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
## Jeder Pod sollte einen "istio-proxy" Container haben
```

---

### Schritt 4: Canary Upgrade durchführen

#### 4.1 Neue Istiod-Revision installieren

```bash
helm install istiod-1-23-0 istio/istiod \
  -n istio-system \
  --version 1.23.0 \
  --set revision=1-23-0 \
  --wait
```

Jetzt laufen **zwei** Istiod-Instanzen parallel:

```bash
kubectl get pods -n istio-system -l app=istiod
## NAME                             READY   STATUS
## istiod-1-22-0-xxxxxxxxx-xxxxx    1/1     Running
## istiod-1-23-0-yyyyyyyyy-yyyyy    1/1     Running
```

#### 4.2 (Optional) Canary-Tag für Testphase

```bash
## Canary-Tag für einzelne Test-Namespaces
istioctl tag set canary --revision 1-23-0

## Test-Namespace umschalten
kubectl label namespace test-ns istio.io/rev=canary --overwrite
kubectl rollout restart deployment -n test-ns
```

Hier testen, ob alles mit der neuen Version funktioniert.

#### 4.3 Prod-Tag auf neue Revision schwenken

```bash
## Tag "prod" zeigt jetzt auf 1-23-0
istioctl tag set prod --revision 1-23-0 --overwrite

## Prüfen
istioctl tag list
## TAG      REVISION
## prod     1-23-0
## canary   1-23-0
```

#### 4.4 Workloads neu starten

Die Namespace-Labels bleiben unverändert (`istio.io/rev=prod`). Neue Pods bekommen automatisch den Sidecar der neuen Revision. Bestehende Pods müssen neu gestartet werden:

```bash
## Alle Deployments im Namespace neu starten
kubectl rollout restart deployment -n default

## Warten bis alle Pods ready sind
kubectl rollout status deployment -n default
```

#### 4.5 Sidecar-Version prüfen

```bash
## Proxy-Version aller Pods anzeigen
istioctl proxy-status
## NAME                    CLUSTER   CDS   LDS   EDS   RDS   ECDS   ISTIOD                          VERSION
## productpage-v1-...      ...       SYNCED ...                      istiod-1-23-0-yyyyy.istio-system 1.23.0
```

---

### Schritt 5: Alte Revision entfernen

Erst wenn **alle** Workloads auf der neuen Revision laufen:

```bash
## Prüfen ob noch Proxies an alter Revision hängen
istioctl proxy-status | grep 1-22-0
## Sollte leer sein!

## Canary-Tag entfernen (falls erstellt)
istioctl tag remove canary

## Alte Istiod-Revision deinstallieren
helm uninstall istiod-1-22-0 -n istio-system

## Altes Gateway upgraden (falls installiert)
helm upgrade istio-ingressgateway istio/gateway \
  -n istio-ingress \
  --version 1.23.0 \
  --set revision=1-23-0

## Istio Base CRDs updaten
helm upgrade istio-base istio/base \
  -n istio-system \
  --version 1.23.0
```

---

### Rollback

Falls Probleme auftreten, Tag einfach zurückschwenken:

```bash
## Tag zurück auf alte Revision
istioctl tag set prod --revision 1-22-0 --overwrite

## Pods neu starten
kubectl rollout restart deployment -n default

## Neue Revision wieder entfernen
helm uninstall istiod-1-23-0 -n istio-system
```

---

### Zusammenfassung der Kommandos

| Schritt | Kommando |
|---------|----------|
| Neue Revision installieren | `helm install istiod-1-23-0 istio/istiod --set revision=1-23-0` |
| Tag umschalten | `istioctl tag set prod --revision 1-23-0 --overwrite` |
| Pods neu starten | `kubectl rollout restart deployment -n <namespace>` |
| Status prüfen | `istioctl proxy-status` |
| Alte Revision entfernen | `helm uninstall istiod-1-22-0` |
| Rollback | `istioctl tag set prod --revision 1-22-0 --overwrite` |

## Prinzip: Side-Car Injection

### Istio Sidecar-Injection


### 1. Verzeichnis anlegen
```bash
cd
mkdir -p ~/manifests/nginx
````

### 2. Nginx-Deployment erstellen

```bash
cat <<'EOF' > ~/manifests/nginx/nginx.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-istio
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx-istio
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: nginx-istio
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF
```

### 3. Sidecar injizieren und anwenden

```bash
kubectl apply -f <(istioctl kube-inject -f ~/manifests/nginx/nginx.yaml)
```

### 4. Injection prüfen

```bash
kubectl get pods -n nginx-istio
```

Erwartetes Ergebnis: `READY 2/2`



## Demo-App (Gateway API) 

### Istio demo-app *bookinfo* installieren (Gateway API - Variante)


### Überblick 

<img width="992" height="615" alt="image" src="https://github.com/user-attachments/assets/5773ce04-fd83-45a6-9914-d2b1b72c1505" />

### Vorbereitung

#### Gateway API - CRD's installieren (Stand 2026-03-12)

   * falls nicht bereits vorher geschehen

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```


```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

### bookdemo app ausrollen 


```
kubectl -n bookinfo apply -f  ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo get all 
```

### testen ob die app funktioniert 

```
kubectl -n bookinfo exec "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

### App mit gateway api nach aussen öffnen 

```
## That's what we do ....
cat  ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl -n bookinfo get gateways
kubectl -n bookinfo get httproutes -o yaml 
```

```
## note the external-ip from this output
## gateway automatically creates a service 
kubectl -n bookinfo get svc bookinfo-gateway-istio
```

```
http://<external-ip>/productpage 
## or in your browser
```



## CNI-Plugin istio

### Vorteile und Installation mit helm


### Warum das Istio CNI Plugin?

Das Istio CNI Plugin ersetzt den `istio-init` Init-Container, der normalerweise die iptables-Regeln für die Traffic-Umleitung zum Envoy Sidecar setzt.

#### Vorteile

- **Keine erhöhten Privileges im Pod** — `NET_ADMIN` und `NET_RAW` Capabilities entfallen, da die iptables-Regeln auf Node-Ebene gesetzt werden
- **Kompatibel mit `restricted` Pod Security Standard** — wichtig für sicherheitskritische Umgebungen
- **Keine Race Conditions** — Traffic-Regeln stehen bereits bevor der erste Container im Pod startet. Ohne CNI Plugin können Init-Container, die vor `istio-init` laufen, Traffic am Envoy vorbeischicken (kein mTLS, keine AuthorizationPolicy)
- **Weniger Ressourcen** — kein zusätzlicher Init-Container-Image-Pull
- **Im Ambient Mode zwingend erforderlich** — wer später auf Ambient Mode migrieren möchte, braucht das CNI Plugin ohnehin

### Voraussetzungen

- Bestehende Istio-Installation via Helm (base + istiod) mit `profile=demo`
- Helm >= 3.x
- `kubectl` Zugriff auf den Cluster
- Das CNI Plugin arbeitet als **Chained CNI Plugin** — es hängt sich an das bestehende CNI an (Flannel, Calico etc.). Bei Cilium-basierten Clustern (z.B. DigitalOcean DOKS, GKE Dataplane v2) vorher Kompatibilität prüfen.

### Schritt-für-Schritt-Anleitung

#### 1. Aktuelle Installation prüfen

```bash
## Installierte Istio Helm Releases anzeigen
helm list -n istio-system

## Aktuelle istiod-Werte prüfen
helm get values istiod -n istio-system
```

Verifiziere, dass bestehende Pods einen `istio-init` Init-Container haben:

```bash
## alle pods im bookinfo - namespace 
kubectl -n bookinfo get pods details-v1-6cc9f5cc44-c5zqf -o jsonpath='{.spec.initContainers[*].name}'
```

Erwartete Ausgabe (vor CNI Plugin): `istio-init`

#### 2. Istio Helm Repo aktualisieren

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

#### 3. Istio CNI Plugin installieren

```bash
helm install istio-cni istio/cni -n istio-system \
  --set profile=demo
```

Warten bis das CNI DaemonSet auf allen Nodes läuft:

```bash
kubectl rollout status daemonset/istio-cni-node -n istio-system
kubectl -n istio-system get ds istio-cni-node
```

#### 4. istiod upgraden — CNI Plugin aktivieren

```bash
## evtl gibt es sonst Probleme - Fehlermeldung, wenn man den Hook vorher nicht löscht 
kubectl delete validatingwebhookconfiguration istio-validator-istio-system
helm upgrade istiod istio/istiod -n istio-system \
  --set profile=demo \
  --set pilot.cni.enabled=true
```
   * Achtung istiod wird nicht neugestartet, weil
   * die Einstellung pilot.cni.enabled in der configmap  istio-sidecar-injector landet
  * nur diese muss verändert werden
  * Diese configmap wird geladen, wenn ein neuer Pod erstellt wird. Das macht istio über den Kubernetes Api-Server 

```
kubectl -n istio-system get cm istio-sidecar-injector
```

#### 5. Bestehende Workloads neu starten

Bestehende Pods haben noch den alten `istio-init` Init-Container. Erst nach einem Restart werden sie ohne Init-Container und mit CNI Plugin Traffic-Umleitung gestartet.

```bash
## Alle Deployments in einem Namespace neu starten
kubectl rollout restart deployment -n bookinfo 

## Oder gezielt ein einzelnes Deployment
kubectl rollout restart deployment/<DEPLOYMENT_NAME> -n bookinfo 
```

#### 6. Verifizierung

**CNI DaemonSet läuft auf allen Nodes:**

```bash
kubectl get daemonset istio-cni-node -n istio-system
```

**Neue Pods haben keinen `istio-init` Init-Container mehr:**

```bash
kubectl get pod <NEUER_POD> -n bookinfo -o jsonpath='{.spec.initContainers[*].name}'
```

Erwartete Ausgabe: istio-validation istio-proxy
  * istio-validation überprüft ob ide iptables - Regeln richtig gesetzt wurden 

**Traffic funktioniert weiterhin:**

```bash
## Aus einem Pod heraus testen
## istio-proxy hat curl mit drauf 
kubectl -n bookinfo exec deploy/productpage-v1 -c istio-proxy -- curl -s http://reviews:9080/reviews/1r.local
```

### Troubleshooting

#### CNI DaemonSet startet nicht

```bash
kubectl describe daemonset istio-cni-node -n istio-system
kubectl logs -l k8s-app=istio-cni-node -n istio-system
```

Häufige Ursache: CNI-Bin- oder Conf-Verzeichnis stimmt nicht mit dem Cluster-Setup überein. Kann über Helm Values angepasst werden:

```bash
helm upgrade istio-cni istio/cni -n istio-system \
  --set profile=demo \
  --set cni.cniBinDir=/opt/cni/bin \
  --set cni.cniConfDir=/etc/cni/net.d
```

#### Pods haben nach Restart immer noch `istio-init`

Prüfen, ob `pilot.cni.enabled=true` gesetzt ist:

```bash
helm get values istiod -n istio-system | grep cni
```

#### Netzwerk-Konnektivität unterbrochen nach CNI Installation

Rollback möglich:

```bash
helm upgrade istiod istio/istiod -n istio-system \
  --set profile=demo \
  --set pilot.cni.enabled=false

kubectl rollout restart deployment -n bookinfo

helm uninstall istio-cni -n istio-system
```

## Steuerung des Netzwerkverhaltens in Istio

### Istio Ingress Gateway vs. Gateway API


### Istio Ingress-Gateway-Flow (bookinfo/productpage) 

![Flow](/images/screenshot_20260314_142715.png)

### Gateway API -> Gateway Flow (bookinfo/productpage)

![Flow](/images/istio/gateway-api-flow.png)

 * Achtung:  VirtualService und DestinationRule werden teilweise immer noch benötigt, auch wenn man mit der Gateway API arbeitet,
     um bestimmte Feature abzubilden  

### Bild-Quelltext (Istio)

```
graph LR
    Client[Client/Browser]
    IG[Istio IngressGateway<br/>istio-ingressgateway Pod]
    GW[Gateway Resource<br/>istio.networking.v1beta1]
    VS[VirtualService<br/>routing rules]
    SvcA[Service A]
    SvcB[Service B]
    
    Client -->|HTTP/HTTPS| IG
    IG -.->|references| GW
    GW -.->|bound to| VS
    VS -->|route to| SvcA
    VS -->|route to| SvcB
    
    style IG fill:#e1f5ff
    style GW fill:#fff3cd
    style VS fill:#fff3cd
```

### Bild-Quelltext (Kubernetes Gateway API)

```
graph TB
    Client[Client/Browser]
    GWC[GatewayClass<br/>infrastructure.cluster.x-k8s.io]
    GW[Gateway<br/>gateway.networking.k8s.io]
    HR[HTTPRoute<br/>routing rules]
    SvcA[Service A]
    SvcB[Service B]
    
    Client -->|HTTP/HTTPS| GW
    GW -.->|instance of| GWC
    HR -.->|attached to| GW
    HR -->|route to| SvcA
    HR -->|route to| SvcB
    
    style GWC fill:#d4edda
    style GW fill:#d4edda
    style HR fill:#d4edda
```

### Gateway API als Nachfolger von IngressGateway (Istio), VirtualService und DestinationRule


### Wichtig Hinweis:

  * In Istio werden immer mehr Sachen von der Gateway API übernommen
  * Aber noch nicht alles ist migriert, deswegen brauche ich telweise immer noch die Objekte (VirtualService und DestinationRule für bestimmte Features

### Was Gateway API ersetzt 

### Kurze Antwort: Für Routing nein, für einige Istio-spezifische Features ja.

#### Was Gateway API vollständig ersetzt (Stand: 03/2026)

| Feature | VirtualService | Gateway API Equivalent |
|---|---|---|
| Request Routing (Header, Path, Method) | ✅ | HTTPRoute `matches` |
| Traffic Splitting / Canary | ✅ | HTTPRoute `backendRefs` mit `weight` |
| URL Rewrite | ✅ | HTTPRoute `URLRewrite` Filter |
| Request Header Modification | ✅ | HTTPRoute `RequestHeaderModifier` |
| Redirects | ✅ | HTTPRoute `RequestRedirect` |
| Request Mirroring | ✅ | HTTPRoute `RequestMirror` |
| gRPC Routing | ✅ | GRPCRoute |

#### Was du weiterhin als Istio-API brauchst

**DestinationRule** — kein Gateway API Equivalent für:
- Circuit Breaking / Outlier Detection
- Connection Pool Settings
- Load Balancing Algorithmus (LEAST_REQUEST, CONSISTENT_HASH)
- TLS-Modus zum Backend

**VirtualService** — noch nötig für:
- **Fault Injection** (Delay, Abort) → gibt es in Gateway API nicht
- **Retries** (Anzahl, Timeout, retryOn Conditions)
- **Timeouts** (request-level)
- **CORS Policy**
- **Regex-basiertes URI-Matching** (Gateway API kann nur Exact und PathPrefix)

#### Praxis-Empfehlung

```
Neues Projekt:
  Routing            → Gateway API (HTTPRoute)
  Fault Injection    → VirtualService (solange nötig)
  Backend-Policies   → DestinationRule
  
Bestehendes Projekt:
  Migration schrittweise, beides kann parallel existieren
```

**Achtung:** Nicht beide gleichzeitig (httpRoute und VirtualService) auf denselben Service anwenden — das führt zu undefiniertem Verhalten. Entweder HTTPRoute **oder** VirtualService pro Service, nicht mischen.



### Virtual Services, Destination Rules


### Überblick

| Ressource | Aufgabe |
|-----------|---------|
| **VirtualService** | *Wohin* wird Traffic geroutet? (Routing-Regeln, Matching, Rewrites) |
| **DestinationRule** | *Wie* genau geht der Traffic? (Subsets, Load Balancing, TLS, Connection Pools) |

Beide arbeiten zusammen: Der VirtualService wählt das Ziel, die DestinationRule definiert die Eigenschaften des Ziels.

---

### VirtualService

Ein VirtualService fängt Traffic ab und leitet ihn anhand von Regeln weiter.

#### Routing-Möglichkeiten

| Feature | Beschreibung |
|---------|-------------|
| Traffic Splitting (Canary) | Über `weight` auf mehrere Subsets verteilen |
| Header-basiertes Routing | `match.headers` mit `exact`, `prefix`, `regex` |
| URI-basiertes Routing | `match.uri` mit `exact`, `prefix`, `regex` |
| Fault Injection | `fault.delay` und `fault.abort` mit Prozentwerten |
| Timeouts & Retries | `timeout`, `retries.attempts`, `retries.perTryTimeout`, `retries.retryOn` |

---

### DestinationRule

Eine DestinationRule definiert Policies **nach** dem Routing (Subsets, Load Balancing, Circuit Breaking, TLS).

#### Traffic Policies

| Feature | Feld | Optionen |
|---------|------|----------|
| Load Balancing | `trafficPolicy.loadBalancer.simple` | `ROUND_ROBIN`, `RANDOM`, `LEAST_CONN`, `PASSTHROUGH` |
| Consistent Hashing | `trafficPolicy.loadBalancer.consistentHash` | `httpHeaderName`, `httpCookie`, `useSourceIp` |
| Circuit Breaking | `trafficPolicy.connectionPool` | `tcp.maxConnections`, `http.http1MaxPendingRequests`, `http.http2MaxRequests` |
| Outlier Detection | `trafficPolicy.outlierDetection` | `consecutive5xxErrors`, `interval`, `baseEjectionTime`, `maxEjectionPercent` |
| mTLS | `trafficPolicy.tls.mode` | `DISABLE`, `SIMPLE`, `MUTUAL`, `ISTIO_MUTUAL` |

Subset-spezifische `trafficPolicy` überschreibt die globale Policy nur für das jeweilige Subset.

---

### Zusammenspiel

1. **DestinationRule** erstellen → Subsets definieren (z.B. `v1`, `v2`, `v3`) + globale Policies (TLS, LB)
2. **VirtualService** erstellen → Routing-Regeln mit Verweis auf die Subsets

---

### Cheat Sheet

| Was willst du? | Ressource | Feld |
|----------------|-----------|------|
| Canary Deployment | VirtualService | `weight` |
| A/B Testing (Header) | VirtualService | `match.headers` |
| Fehler simulieren | VirtualService | `fault` |
| Timeouts setzen | VirtualService | `timeout` / `retries` |
| Subsets (v1/v2) | DestinationRule | `subsets` |
| Load Balancing | DestinationRule | `trafficPolicy.loadBalancer` |
| Circuit Breaker | DestinationRule | `outlierDetection` |
| mTLS | DestinationRule | `trafficPolicy.tls` |
| Connection Limits | DestinationRule | `connectionPool` |

---

## Übungen Steuerung des Netzwerkverhaltens (Gateway API)

### Übung: Header-basiertes Routing


### Prerequisites

  * Bookinfo - Projekt aufgesetzt.

### Schritt 1: Vorbereitung: Status review-pods 

  * Status: alle Pods sind unter einem Service erreichbar

```
## Es gibt 3 verschieden review-pods (v1, v2, v3) 
kubectl -n bookinfo get pods --show-labels | grep review
```

```
## Ein Service zeigt auf alle pods (Alle versionen der Review - Pods)
kubectl -n bookinfo get svc | grep reviews  
```

### Schritt 2: Vorher (ohne request routing) 

  * Es werden alle Pods angezeigt, die das Label: app:reviews haben
  * D.h. jedesmal wenn ich die Seite öffne, wird eine andere Version angegezeigt (v1, v2 oder v3) - * d.h. es werden ganz normal die Services von Kubernetes verwendet **
  * Service (selector: app:reviews)

```
kubectl -n bookinfo get svc reviews -o yaml
kubectl -n bookinfo get pods -l app=reviews --show-labels
```

```
## Gateway wurde in der Übung vorher angelegt
## du findest so die IP des gateways raus 
kubectl -n bookinfo get gateway
```

```
GATEWAY_URL=<ip-aus-der-vorigen-Ausgabe-eintragen>
```

```
## Im Browser mehrmals ausführen
## Im Block mit den Reviews wechselt die Version 
$GATEWAY_URL/productpage
```


### Schritt 3: Übung (jetzt request - routing) 

**Voraussetzung:**

- Bookinfo-App läuft bereits im Namespace `bookinfo`
- Service Reviews existiert 
- Es gibt 3 verschieden Pods an Reviews (v1, v2 und v3)
- Ingress/Gateway + `GATEWAY_URL` (IP: http://164.90.237.35/productpage aus der vorherigen Übung vorhanden

#### 0. Vorbereitung

```bash
mkdir -p ~/manifests/requests
cd ~/manifests/requests

## Die Service-Versionen anlegen
cp -a ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml bookinfo-versions.yaml
kubectl -n bookinfo apply -f .
```

---

#### 1. HTTPRoute: Alle Requests → `reviews-v1`

```bash
cat <<EOF > ~/manifests/requests/httproute-reviews-v1.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF

kubectl apply -f httproute-reviews-v1.yaml
kubectl -n bookinfo get httproute reviews -n bookinfo
```

```
## Vergleich mit allen httproutes
kubectl -n bookinfo get httproute
## booking for north - south traffik
```

```
## Anzeige im Browser - es ist immer die v1
http://164.90.237.35/productpage
```


---

#### 2. HTTPRoute anpassen: User `jason` → `reviews-v2`, Rest → `reviews-v1`

```bash
cat <<EOF > ~/manifests/requests/httproute-reviews-jason-v2.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - matches:
    - headers:
      - name: end-user
        value: jason
    backendRefs:
    - name: reviews-v2
      port: 9080
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF

kubectl apply -f httproute-reviews-jason-v2.yaml
kubectl -n bookinfo get httproute reviews -n bookinfo -o yaml
```

---

### 3. Testen im Browser

```bash
echo "$GATEWAY_URL"
## Beispiel: http://<IP>:<PORT>

## 1. Im Browser: $GATEWAY_URL/productpage aufrufen (nicht eingeloggt oder anderer User)
##    → Reviews ohne Sterne (v1)

## 2. Im Browser: als User "jason" einloggen
##    → Reviews mit Sternen (v2)
```

---

### 4. Aufräumen

```bash
kubectl delete -f httproute-reviews-v1.yaml --ignore-not-found
kubectl delete -f httproute-reviews-jason-v2.yaml --ignore-not-found
```

### Reference: 

  * https://istio.io/latest/docs/examples/bookinfo/#define-the-service-versions

### Übung: Traffic-Shifting / Load-Balancing


  * Schrittweise Umleitung von Netzwerk-Traffic zwischen zwei Service-Versionen


#### 0. Vorbereitung

```bash
mkdir -p ~/manifests/traffic-shifting
cd ~/manifests/traffic-shifting 

## Die Service-Versionen anlegen
cp -a ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml bookinfo-versions.yaml
kubectl -n bookinfo apply -f .
```
#### 1. 100% Traffic -> reviews.v1 

```
cat <<'EOF' > ~/manifests/traffic-shifting/route-reviews-v1.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
EOF
```

```
kubectl apply -n bookinfo -f route-reviews-v1.yaml
kubectl get httproute -n bookinfo reviews -o yaml | less
```

```
## Status ist interessant !
```

<img width="748" height="172" alt="image" src="https://github.com/user-attachments/assets/f1c928ea-0819-4a42-8902-e6137c46438c" />



#### 2. Testen 

```
## Seite öffnen
http://<deine-ip>/productpage

## Egal wie oft du die Seite lädst, es bleibt immer v1
```

#### 3. 50% (v1) /50% (v3) Traffic

```
cat <<'EOF' > ~/manifests/traffic-shifting/route-reviews-50-50.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
      weight: 50
    - name: reviews-v3
      port: 9080
      weight: 50
EOF
```

```bash
kubectl apply -n bookinfo -f route-reviews-50-50.yaml
kubectl get httproute -n bookinfo reviews -o yaml | head -n 40
```

### 4. Testen 


```
## Seite öffnen
http://<deine-ip>/productpage

## Abwechselnd bei mehrmals laden v1 (keine Sterne) und v3 (sterne)
```

### 5. 100% auf v3 

```
cat <<'EOF' > ~/manifests/traffic-shifting/route-reviews-v3.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
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
EOF
```

```
kubectl apply -n bookinfo -f route-reviews-v3.yaml
kubectl get httproute -n bookinfo reviews -o yaml | head -n 50
```

<img width="749" height="434" alt="image" src="https://github.com/user-attachments/assets/037de8c8-a50d-40fa-bd58-41a6719e3f25" />


```
## Seite öffnen
http://<deine-ip>/productpage

## bei mehrmals laden immer v3
```

#### 6. Aufräumen 

```
kubectl delete -n bookinfo httproute reviews --ignore-not-found
```

### Reference:

 * https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/

### Pfadbasiertes Routing


```
## Pfadbasiertes Routing ist bereits eingerichtet
## Achtung: Nord/Süd - Traffik / httproute zeigt auf das Gateway 
kubectl -n booking get httproute bookinfo -o yaml
```

### Übung: LoadBalancing (LEAST_CONN) - Regeln


### Ziel

Den Unterschied zwischen LEAST_CONN und ROUND_ROBIN sichtbar machen. Dafür deployen wir einen Service mit schnellen und einem langsamen Pod. Bei parallelen Requests zeigt LEAST_CONN messbar besseres Verhalten.

### Voraussetzungen

- Kubernetes-Cluster mit Istio (inkl. Gateway API CRDs)
- `kubectl` und `istioctl` konfiguriert


### Schritt 0: traefik deinstallieren 

  * Wir haben aktuelle keine freien IP's. Traefik ist installiert und belegt eine IP, brauchen wir aber aktuell nicht

```
helm list -A | grep -i traefik
helm -n ingress uninstall traefik
```


---

### Schritt 1: Namespace vorbereiten

```bash
kubectl create namespace lb-demo
kubectl label namespace lb-demo istio-injection=enabled
```

---

### Schritt 2: Backend deployen (schnelle + langsamer Pod)

Alle Pods laufen hinter dem gleichen Service. Der einzige Unterschied: der langsame Pod wartet 2 Sekunden vor jeder Antwort.

```
cd 
mkdir -p manifests/lb-least-conn
cd manifests/lb-least-conn
```

```
nano backend.yaml
```


```yaml
## backend.yaml
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

### Schritt 3: Gateway + HTTPRoute

```
nano gateway.yaml
```

```yaml
## gateway.yaml
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

### Schritt 4: DestinationRule mit LEAST_CONN

```
nano destination-rule.yaml
```

```yaml
## destination-rule.yaml
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

### Schritt 5: Envoy-Konfiguration prüfen

```
GWPOD=$(kubectl get pods -n lb-demo -l gateway.networking.k8s.io/gateway-name=lb-gateway -o jsonpath='{.items[0].metadata.name}')
```


```
istioctl proxy-config cluster $GWPOD -n lb-demo   --fqdn "backend.lb-demo.svc.cluster.local" -o yaml | grep "LEAST"
## expected output:
##  lbPolicy: LEAST_REQUEST
```

### Schritt 6: Test-Funktion definieren

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

### Schritt 7: LEAST_CONN vs ROUND_ROBIN vergleichen

#### Test mit LEAST_CONN

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

#### Umschalten auf ROUND_ROBIN

```bash
kubectl patch destinationrule backend -n lb-demo --type merge \
  -p '{"spec":{"trafficPolicy":{"loadBalancer":{"simple":"ROUND_ROBIN"}}}}'
```

```
## Unbedingt warten, bis keine Ausgabe LEAST_ mehr kommt
istioctl proxy-config cluster $GWPOD -n lb-demo   --fqdn "backend.lb-demo.svc.cluster.local" -o yaml | grep "lbPolicy"
```

```
sleep 2

echo "=== ROUND_ROBIN ==="
test_lb
```

Erwartete Ausgabe:

```
--- Sende 30 Requests (10 parallel) ---
--- Gesamtdauer: x?s ---
--- Verteilung pro Pod ---
     10 backend-fast-abc123 delay=0s
     10 backend-fast-def456 delay=0s
     10 backend-slow-xyz789 delay=2s
```

> ROUND_ROBIN verteilt stur gleichmäßig – der langsame Pod bekommt genauso viele Requests. Die Gesamtdauer ist deutlich höher.

#### Zurück auf LEAST_CONN

```bash
kubectl patch destinationrule backend -n lb-demo --type merge \
  -p '{"spec":{"trafficPolicy":{"loadBalancer":{"simple":"LEAST_CONN"}}}}'
```

---

### Schritt 8: Ergebnis

| Metrik | LEAST_CONN | ROUND_ROBIN |
|---|---|---|
| **Requests an langsamen Pod** | wenige (~3-5) | gleichmäßig (~10) |
| **Gesamtdauer (30 Req)** | ~4s | ~8s |
| **Prinzip** | Bevorzugt freie Pods | Verteilt stur reihum |

LEAST_CONN erkennt, dass der langsame Pod noch beschäftigt ist (offene Verbindung), und schickt neue Requests an die freien, schnellen Pods.

---

### Schritt 9: Wann welchen Algorithmus?

| Algorithmus | Envoy-Name | Wann sinnvoll? |
|---|---|---|
| **ROUND_ROBIN** | `ROUND_ROBIN` | Default. Gleichförmige, kurze Requests |
| **LEAST_CONN** | `LEAST_REQUEST` | Variable Latenzen, Long-Running Connections (gRPC, WebSocket) |
| **RANDOM** | `RANDOM` | Große Cluster, vermeidet Hot-Spots |
| **PASSTHROUGH** | – | Kein LB, direkt an Upstream-IP |

---

### Bonusaufgabe: HTTPRoute East-West mit Service parentRef

Seit der GAMMA-Initiative unterstützt Istio `HTTPRoute` auch für **East-West Traffic** (Service-zu-Service). Dafür setzt man `parentRefs` auf einen **Service** statt auf einen Gateway.

#### Client-Pod deployen

```
nano client.yaml
```

```yaml
## client.yaml
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

#### Zweiten Backend-Service anlegen

```yaml
## backend-fast-svc.yaml
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

#### HTTPRoute: East-West Traffic Split

Alle Calls an `backend:80` werden zu 100% an die schnellen Pods umgeleitet:

```yaml
## httproute-east-west.yaml
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

#### Testen aus dem Client-Pod

```bash
CLIENT=$(kubectl get pods -n lb-demo -l app=client -o jsonpath='{.items[0].metadata.name}')

## Vorher: alle Pods (inkl. slow) erreichbar
## Nachher: nur noch fast-Pods antworten
for i in $(seq 1 10); do
  kubectl exec -n lb-demo $CLIENT -- curl -s http://backend/
done
```

> Alle Antworten kommen nur noch von `backend-fast` Pods – der langsame Pod ist per East-West HTTPRoute komplett aus dem Routing genommen.


### Test von aussen 

  * Das geht auch !

```
curl -s http://$GATEWAY_IP/lb-test
```


### Aufräumen

```bash
kubectl delete namespace lb-demo
```

---

### Zusammenfassung

| Ebene | Ressource | parentRef | Aufgabe |
|---|---|---|---|
| **North-South** | `Gateway` + `HTTPRoute` | `kind: Gateway` | Externer Eintrittspunkt + Routing |
| **East-West** | `HTTPRoute` | `kind: Service` | Service-zu-Service Traffic Split |
| **LB-Algorithmus** | `DestinationRule` | – | LEAST_CONN, RANDOM, Consistent Hash |

**Merke:**
- `HTTPRoute` kann beides: North-South (`parentRef` → Gateway) **und** East-West (`parentRef` → Service)
- Für East-West setzt man `kind: Service` + `group: ""` in `parentRefs`
- `DestinationRule` definiert den LB-Algorithmus – unabhängig von der Routing-Ressource

## Service Resilience & Fehlertoleranz (Gateway API)

### Circuit Breaker


### Aufgabe des Circuit Breakers:

```
Der Circuit Breaker in Istio schützt Deine Services davor, einen überlasteten oder fehlerhaften Backend-Service weiter mit Requests zu bombardieren.
```

### Wie ?

Du konfigurierst im `DestinationRule` unter `trafficPolicy.connectionPool` und `outlierDetection` zwei Dinge:

1. **Connection Pool Limits** (`connectionPool`) — begrenzt die Anzahl gleichzeitiger TCP-Connections und wartender HTTP-Requests. Wird das Limit überschritten, werden neue Requests sofort mit 503 abgelehnt statt endlos zu warten.

1. **Outlier Detection** (`outlierDetection`) — erkennt ungesunde Endpoints. Wenn ein Pod z.B. 5× hintereinander 5xx zurückgibt, wird er temporär aus dem Load-Balancing-Pool entfernt ("evicted"). Nach einer Pause darf er wieder rein.

### Voraussetzungen

* Istio ist installiert (Sidecar Mode).
* Namespace `bookinfo` ist mit automatischer Sidecar-Injection gelabelt:

```bash
kubectl label namespace bookinfo istio-injection=enabled --overwrite
```

* Das Istio-Repo liegt unter `~/istio` (Samples unter `~/istio/samples/...`).

---

### 1️⃣ Arbeitsverzeichnis anlegen

```bash
mkdir -p ~/manifests/circuit-breaker
cd ~/manifests/circuit-breaker
```

---

### 2️⃣ httpbin im Namespace `bookinfo` deployen

```bash
kubectl apply -n bookinfo -f ~/istio/samples/httpbin/httpbin.yaml
kubectl get pods -n bookinfo -l app=httpbin
```

Warte, bis der Pod `Running` ist.

---

### 3️⃣ DestinationRule mit Circuit Breaker anlegen

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

  * Erklärung TCP/http
  * Request 1 → bekommt die eine Connection → wird verarbeitet
  * Request 2 → Connection belegt → geht in die Pending Queue (1 Platz)
  * Request 3 → Connection belegt, Queue voll → sofort 503

```
consecutive5xxErrors: 1 — Ein einziger 5xx-Fehler reicht, und der Endpoint fliegt raus. In Produktion eher 5–10, damit nicht ein einzelner Timeout sofort zur Eviction führt.

interval: 1s — Envoy prüft jede Sekunde, ob ein Endpoint auffällig ist. Das ist der Auswertungszyklus. Default wäre 10s. Heißt: nach einem 5xx dauert es maximal 1 Sekunde, bis der Pod evicted wird.

baseEjectionTime: 3m — Ein evicted Endpoint bleibt 3 Minuten draußen, bevor er wieder in den Pool darf. Wichtig: bei wiederholter Eviction multipliziert sich die Zeit — beim zweiten Mal 6 Minuten, beim dritten 9 Minuten usw. Das bestraft "Wiederholungstäter".

maxEjectionPercent: 100 — Es dürfen alle Endpoints gleichzeitig evicted werden. Default ist 10%. Mit 100% kann es passieren, dass der Service komplett unerreichbar wird, weil kein einziger Endpoint mehr im Pool ist. In Produktion gefährlich — da willst du immer mindestens einen Endpoint behalten. Für die Demo ist es aber praktisch, weil du klar zeigen kannst: ein Fehler → Pod fliegt raus → alles 503.
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

### 4️⃣ Fortio-Client im Mesh deployen

 * Fortio ist ein Open-Source Load-Testing-Tool,

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

### 5️⃣ Sanity-Check: Ein einzelner Request

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio curl -quiet http://httpbin:8000/get
```

Erwartung: **HTTP 200 OK** mit JSON-Antwort.

---

### 6️⃣ Circuit Breaker „anrucken“ (2 Verbindungen)

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio --   /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning   http://httpbin:8000/get 2>&1 | grep "Code "
```

* `-c 2` → 2 gleichzeitige Verbindungen
* `-n 20` → 20 Requests

Erwartung: Die meisten Requests sind 200, einige ggf. 503.

---

### 7️⃣ Circuit Breaker deutlich auslösen (3 Verbindungen)

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c fortio -- \
  /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning \
  http://httpbin:8000/get 2>&1 | grep "Code "
```

Erwartung: **Deutlich mehr 503** (Circuit Breaking greift).

---

### 8️⃣ Envoy-Stats im Sidecar prüfen

```bash
kubectl exec -n bookinfo "$FORTIO_POD" -c istio-proxy -- \
  pilot-agent request GET stats | grep httpbin.bookinfo.svc.cluster.local | grep pending
```

Achte auf Werte wie `upstream_rq_pending_overflow` → zeigt an, wie viele Requests wegen Circuit Breaking abgewiesen wurden.

---

### 9️⃣ Aufräumen

```bash
## Circuit-Breaker-Regel entfernen
kubectl delete -f dr-httpbin-circuit-breaker.yaml

## Fortio-Client entfernen
kubectl delete -n bookinfo -f ~/istio/samples/httpbin/sample-client/fortio-deploy.yaml

## httpbin entfernen
kubectl delete -n bookinfo -f ~/istio/samples/httpbin/httpbin.yaml
```

---

### Rate Limiting


### Walkthrough 

  * Bind EnvoyFilter to workload app=productpage

```
cd
mkdir -p manifests/local-rate-limit
cd manifests/local-rate-limit
## Get gateway ip 
kubectl -n bookinfo get gateway bookinfo-gateway
GATEWAY_URL="http://<your-ip-of-gateway>"
```

```
nano rate-limit-240-60s-ok.yaml
```

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-local-ratelimit-svc
  namespace: bookinfo
spec:
  workloadSelector:
    labels:
      app: productpage
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          portNumber: 9080 # Port of Service / best practice -> be specific
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              enable_x_ratelimit_headers: DRAFT_VERSION_03 # mehr infos im outpunt header - wieviele token sind verbraucht 
              token_bucket:
                max_tokens: 240
                tokens_per_fill: 240
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              response_headers_to_add:
                - append: false
                  header:
                    key: x-local-rate-limit
                    value: 'true'
```

```
kubectl apply -f rate-limit-240-60s-ok.yaml
```

```
while true; do curl -s "$GATEWAY_URL/productpage" -o /dev/null -w "%{http_code}\n"; done
```

```
## works will out put
```

<img width="87" height="411" alt="image" src="https://github.com/user-attachments/assets/adaa0259-730d-4ba7-aa8d-4e17be50c92b" />

```
## now try with 20 per 60s
```

```
nano rate-limit-20-60s-not.yaml
```

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-local-ratelimit-svc
  namespace: bookinfo
spec:
  workloadSelector:
    labels:
      app: productpage
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          portNumber: 9080 # port of service -> best practice be specific
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 20
                tokens_per_fill: 20
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              response_headers_to_add:
                - append: false
                  header:
                    key: x-local-rate-limit
                    value: 'true'
```

```
kubectl apply -f rate-limit-20-60s-not.yaml
```

```
while true; do curl -s "$GATEWAY_URL/productpage" -o /dev/null -w "%{http_code}\n"; done
```

```
## now we have a lot of too many connections
## rate limit helps to throttle 
```
<img width="88" height="728" alt="image" src="https://github.com/user-attachments/assets/b70f8898-3a48-4569-8fe2-d998cdc50022" />

### Analyse direkt im istio-proxy (sidecar) 

```
istioctl proxy-config listener -n bookinfo deploy/productpage-v1 --port 15006 -o json | grep -A40 local_ratelimit
```

#### Erklärung - Ablauf 

1. Ein Request kommt am Pod an (z.B. auf Port 9080)
2. **iptables-Regeln** (von `istio-init` eingerichtet) fangen den Traffic ab und leiten ihn auf Port **15006** um
3. Der `virtualInbound` Listener auf 15006 nimmt den Request entgegen
4. Envoy schaut sich TLS, ALPN und den **Original-Zielport** an (via `original_dst` Listener Filter)
5. Basierend darauf wird die passende **Filter Chain** ausgewählt (z.B. `0.0.0.0_9080` mTLS)
6. Der Request durchläuft die HTTP Filter (inkl. Rate Limiting) und wird an den lokalen App-Container weitergeleitet


### Chaos Engineering mit Istio

  * https://istio.io/latest/docs/examples/microservices-istio/production-testing/

## Service Resilience & Fehlertoleranz (nur VirtualService)

### Bemerkungen zu nur VirtualService und nicht Gateway API - httproute


  * Some features are only implemented in VirtualService (like FaultInjection, Retries)

### Vorgehensweise 

   * httpRoute wird für NorthSouth Traffic verwendet (das was ins Cluster reinkommt)
   * Im Mesh kann ich entweder httpRoute oder virtualService verwenden
   * Wenn eine Feature nur in virtualService und noch nicht für die Gateway Api implementiert ist, muss ich virtualService im Mesh verwenden (Beispiel Fault Injection, Retries)

### Warum kann ich im Mesh httpRoute und und VirtualService nicht mischen 

   * Um nicht alles neu zu entwickeln, hat sich istio httpRoute (Mesh) -> in -> virtualService umzuschreiben
   * D.h. für einen Service entweder virtualService oder httpRoute
   * Wann immer möglich (weil alle Feature, die ich brauche drin sind), würde ich httpRoute nehmen 

### Retries VirtualService only !


  * Retries in der Gateway API noch nicht implementiert
  * Wenn benötigt, dann für den Service (intern-mesh)-> keine httproute verwenden, sondern VirtualService
  * So haben wir es hier auch implementiert 

### Wofür

  * Wenn ein Pod nicht erreicht werden kann, und ein anderer HTTP-Code als 200 zurückkommt, dann nochmal probieren 

### Vorbereitung: Test-Server erstellen, der 500er erstellt 

```
cd
mkdir -p manifests/retry-flaky
cd manifests/retry-flaky
```

```
nano flaky-server.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: flaky-server
  namespace: bookinfo
data:
  server.py: |
    from http.server import BaseHTTPRequestHandler, HTTPServer
    import random

    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            # 50% Chance auf 500, sonst 200
            if random.random() < 0.5:
                self.send_response(500)
                self.end_headers()
                self.wfile.write(b"oops, random 500\n")
            else:
                self.send_response(200)
                self.end_headers()
                self.wfile.write(b"all good, 200\n")

        def log_message(self, format, *args):
            # Einfaches Logging auf STDOUT
            print("%s - - [%s] %s" % (self.client_address[0], self.log_date_time_string(), format % args))

    if __name__ == "__main__":
        port = 8080
        server = HTTPServer(("", port), Handler)
        print(f"Starting flaky server on port {port}")
        server.serve_forever()

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flaky-server
  template:
    metadata:
      labels:
        app: flaky-server
    spec:
      containers:
      - name: flaky-server
        image: python:3.11-alpine
        command: ["python", "/app/server.py"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: server-code
          mountPath: /app
      volumes:
      - name: server-code
        configMap:
          name: flaky-server
          items:
          - key: server.py
            path: server.py
---
apiVersion: v1
kind: Service
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  selector:
    app: flaky-server
  ports:
  - name: http
    port: 8080
    targetPort: 8080
```

```
kubectl apply -f flaky-server.yaml
```


### Schritt 2: VirtualHost ohne retries

  * Hier könnte man auch httproute (east-west) verwenden 

```
nano virtual-no-retries.yaml
```

```
## Istio VirtualService mit Retries für flaky-server
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  hosts:
  - flaky-server
  http:
  - route:
    - destination:
        host: flaky-server
        port:
          number: 8080
```

```
kubectl apply -f virtual-no-retries.yaml
```


#### Schritt 3: Testscript flaky erstellt  und ausführen 

```
touch flaky-test.sh
chmod u+x flaky-test.sh
nano flaky-test.sh
```

```
##!/usr/bin/env bash

set -e

NAMESPACE="bookinfo"
SERVICE="flaky-server"
PORT=8080

REVIEWS_POD=$(kubectl get pod -n "$NAMESPACE" -l app=reviews -o jsonpath='{.items[0].metadata.name}')
echo "Using REVIEWS_POD: $REVIEWS_POD"
echo "Calling http://$SERVICE:$PORT/ ..."
echo "Zum Abbrechen: STRG+C"
echo

kubectl exec -n "$NAMESPACE" -c reviews "$REVIEWS_POD" -- sh -c '
  i=0
  while true; do
    i=$((i+1))
    code=$(curl -s -o /dev/null -w "%{http_code}\n in %{time_total}s" http://flaky-server:8080/ || echo "000 in 0")
    echo "[Request $i] -> $code"
    sleep 1
  done
'
```

```
## 500er beobachte
## Wieviele 500er 
./flaky-test.sh
```

```
CTRL + C 

```

### Schritt 4: virtualservice zu mit retries modifizieren 

```
nano virtualservice-mit-retries.yaml
```

```
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  hosts:
  - flaky-server
  http:
  - route:
    - destination:
        host: flaky-server
        port:
          number: 8080
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: "5xx,connect-failure,refused-stream,reset"
```

```
kubectl apply -f virtualservice-mit-retries.yaml 
```

```
## nochmal testen
## Wieviele 500er jetzt
./flaky-test.sh
```

```
CTRL + C
```

### Verifizieren direkt im Flaky-Pod

  * Er gibt 200 oder 500 zurück 
  * Für alle gleichen Anfragen, gibt es es eine Request_id 

  ![Retries in flaky-server](image-2.png)

```
FLAKY_POD=$(kubectl get pod -n bookinfo -l app=flaky-server -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n bookinfo -c istio-proxy "$FLAKY_POD" 
```



### Aufräumen 

```
kubectl delete -f .
```

### Fehlerinjektion (z.B. 500er) VirtualService only !


  * The gateway api does not support fault-injection (so the http-route object)
  * We are using the gateway api, but can still use the VirtualService for Service-2-Service call inside the mesh

### Walkthrough 

#### Step 1: Prepare 

```
cd
mkdir -p manifests/fault-injection
cd manifests/fault-injection
```

```
cp -a ~/istio/samples/bookinfo/networking/virtual-service-all-v1.yaml . 
cp -a ~/istio/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml .
kubectl -n bookinfo apply -f . 
```

#### Step 2: Fault-Injection - Abort (500er)

```
kubectl apply -n bookinfo -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
EOF
```

#### Step 3: Seite aufrufen und als jason einloggen

  * <dein-ip-des-gateway>/productpage 

<img width="929" height="171" alt="image" src="https://github.com/user-attachments/assets/37b241df-dd02-44c1-bd18-2797b405bac5" />

#### Step 4: Ausloggen oder als andere Nutzer einloggen


  * Reviews gehen wieder 

#### Step 5: Cleanup 

``` 
kubectl delete -f . --ingore-not-found 
```

## Debugging

### Debugging mit debug/run pod


### Why like this ?

 * We need a way that a service is mounted into the pod (service-account is used) 
 * Same serviceAccount that would be used by productpage - pod itself 

### Variante 1: Debug-Container zum Debuggen 

  * Debug Container in productpage - pod starten, um Verbindung zu pod -> Review zu debuggen 

```
kubectl -n bookinfo get pods | grep productpage
## diesen entsprechend hier verwenden 
kubectl -n bookinfo debug productpage-v1-54bb874995-rr7cv -it --image=busybox
```

```
## in der bash 
wget -O - http://reviews:9080/reviews/1
```

```
exit
```

### V2 - Eigener Pod - Podtester  

```
kubectl -n bookinfo run --rm -it podtester --image=busybox --overrides='{ "spec": { "serviceAccount": "bookinfo-productpage" }  }'
```

## Sichere Service-Kommunikation mit istio (Sicherheit)

### Sicherheit & Zero Trust mit Istio


### 1️⃣ **Strong Authentication (mTLS überall)**

* Jeder Pod bekommt automatisch ein **X.509-Zertifikat** (~SpiffeID).
* **Alle** Services sprechen untereinander **mutual TLS**.
* Istio überprüft:

  * Ist der Client wirklich der, der er vorgibt zu sein?
  * Passt die SPIFFE Identity?

👉 *Nichts darf unverschlüsselt, nichts darf anonym miteinander reden.*

---

### 2️⃣ **No implicit trust**

Mit einer deny - policy (oder einer einzigen Allow-Policy) gilt heute in Istio:
**Alles ist DENY, bis du ALLOW definierst.**

D. h. nur weil zwei Services im gleichen Namespace laufen, dürfen sie sich **nicht automatisch** gegenseitig aufrufen.

→ Du definierst explizit **RequestAuthentication** und **AuthorizationPolicy**.

---

### 3️⃣ **Fine-grained Authorization**

Istio entscheidet:

* **Wer** (Service identity / JWT claims)
* **darf** (ALLOW)
* **was** (HTTP-Verb, Pfad, Port)
* **wohin** (Service, Namespace)
* **von wo** (IP, Namespace, Principals)

Beispiel:
„Nur `reviews` darf `/ratings/*` aufrufen – aber nur GET, nicht POST.“

Das ist Zero-Trust.

---

### 4️⃣ **Policy enforcement auf Infrastruktur-Ebene**

Alle Regeln gelten **zentral**, unabhängig vom Code des Services.
Das heißt:

* Keine ACLs mehr im Code
* Keine Bibliotheken anpassen
* Keine Firewall-Regeln auf Node-Ebene

→ Der Sidecar (oder Waypoint) erzwingt die Security.

---

### 5️⃣ **Identity-based Security (nicht IP-basiert)**

In Kubernetes ändern sich IPs permanent → unbrauchbar.

Istio arbeitet stattdessen mit **Identitäten**, z. B.:

```
spiffe://cluster.local/ns/bookinfo/sa/productpage
```

→ Diese Identität wird über mTLS geprüft

---

## 🧠 Merksatz

**Zero-Trust in Istio:
„Authenticate everything, authorize explicitly, trust nobody automatically.“**


### Was ist in istio deep-defense (defense in depth) ?


Istio secures your microservices at **multiple layers**, not just at one point.
Each layer is independent, so a failure in Layer A does *not* break Layer B or C.

Let’s walk through the layers:

## Diagramm 

<img width="579" height="922" alt="image" src="https://github.com/user-attachments/assets/572a7770-661a-4bdd-8562-13aedc238b10" />


---

## **1️⃣ Identity Layer — Strong Workload Identity (SPIFFE/SVID)**

Istio gives each workload a strong cryptographic identity:

* SPIFFE ID: `spiffe://cluster.local/ns/bookinfo/sa/reviews`
* Stored in an mTLS certificate
* Rotated automatically every few minutes/hours

Why this is depth:

✔ Even if the network is compromised, identity is still secure.
✔ Even if someone spoofs an IP, they *cannot spoof* the SPIFFE ID.

---

## **2️⃣ Encryption Layer — Mutual TLS (mTLS)**

Across **all service-to-service traffic**, Istio enforces:

* Encryption
* Authentication (client cert)
* Integrity (no tampering)
* Replay protection

mTLS is handled by the sidecar (or waypoint in Ambient).

Why this is depth:

✔ Even if a pod is compromised, an attacker cannot sniff traffic of other services.
✔ Even if a malicious service sends traffic, it must prove its identity cryptographically.

---

## **3️⃣ Transport Layer — L7-aware Authorization (RBAC at the mesh layer)**

Istio AuthorizationPolicies allow very fine-grained access control:

Examples:

* Only service A may call service B
* Only `GET /api/v1/orders` is allowed
* Only traffic with a specific JWT can reach a workload
* Deny all by default + whitelist exceptions

Why this is depth:

✔ Even if mTLS is on, you *still* get application-layer RBAC.
✔ Even if a token leaks, policies can enforce additional identity checks.

---

## **4️⃣ Credential Layer — JWT Authentication**

Istio allows binding of **end-user identity** (JWT/OIDC) into the mesh:

* Validate JWT
* Verify iss / aud
* Use JWKS URI to fetch signing keys
* Bind user identity to traffic and enforce RBAC

Why this is depth:

✔ Even if an attacker gets inside the cluster, they *still need valid end-user credentials*.
✔ Services cannot impersonate users.

---

## **5️⃣ Network Layer — Ingress & Egress security**

Istio can enforce:

#### Ingress:

* TLS termination
* JWT validation
* Rate limiting
* Web Application Firewall integration (Envoy filters)

#### Egress:

* Strict allowlists
* TLS origination
* Domain-based restrictions

Why this is depth:

✔ Even if an internal service is compromised, it cannot exfiltrate data (egress controls).
✔ Even if the user-facing API is attacked, gateway protections apply before workload.

---

## **6️⃣ Observability Layer — Zero-effort logging + tracing**

Istio gives:

* Distributed tracing
* Access logs (source, destination, identity)
* Metrics per service / route / error code
* mTLS metric visibility

Why this is depth:

✔ Even if someone tries to bypass security, you see clear traces.
✔ Unauthorized or weird traffic stands out immediately.

---

## **7️⃣ Policy Layer — Rate limiting, Fault injection, Circuit breaking**

Istio uses Envoy to enforce:

* Retry budgets
* Local or global rate limits
* Timeouts
* Circuit breakers
* Head-of-line attack protection

Why this is depth:

✔ Even if traffic is valid but malicious (DoS), rate limits protect workloads.
✔ Segments failure domains so one service cannot overload another.

---

## **8️⃣ Workload Layer — Sidecar isolation (Classic Istio Mode)**

In sidecar mode:

* Workloads can’t talk directly to the network
* Envoy controls all outbound and inbound traffic
* Pod cannot bypass mTLS or policies (unless sysadmin-level access)

Why this is depth:

✔ Even if app code is vulnerable, the network guardrail remains intact.
✔ Compromised code ≠ compromised network.

(With Ambient mode, this moves to per-node waypoints; same idea but different architecture.)

---

## **Summary — Istio’s Defense in Depth**

| Layer                    | Purpose                     | Benefit                                |
| ------------------------ | --------------------------- | -------------------------------------- |
| **mTLS**                 | Encryption + authentication | Prevent sniffing + spoofing            |
| **SPIFFE identities**    | Strong workload identity    | Prevent impersonation                  |
| **AuthZ policies**       | L7 firewall                 | Prevent unauthorized access            |
| **JWT Rules**            | End-user identity           | Stop token reuse / user spoofing       |
| **Ingress/Egress rules** | Cluster edge security       | Stop data exfil / limit attack surface |
| **Rate limits / CBs**    | Traffic control             | Prevent overload attacks               |
| **Sidecar isolation**    | Network control             | Prevent bypass                         |
| **Observability**        | Full visibility             | Detect attacks early                   |

Together, these layers form a **mesh-level, enterprise-grade security architecture** — which is exactly what *Defense in Depth* means.

---



### Überblick 3 Säulen -  Sichere Service-Kommunikation


### Überblick

Sichere Kommunikation im Istio Service Mesh basiert auf drei Säulen:

| Säule | Ressource | Funktion |
|-------|-----------|----------|
| Verschlüsselung & Identität | `PeerAuthentication` | mTLS zwischen Services |
| Zugriffskontrolle | `AuthorizationPolicy` | Wer darf mit wem kommunizieren |
| End-User-Authentifizierung | `RequestAuthentication` | JWT-Token-Validierung |

Erst das Zusammenspiel aller drei ergibt ein vollständiges Zero-Trust-Modell.

---

### 1. mTLS mit PeerAuthentication

#### Was passiert unter der Haube?

- Istio stellt jedem Workload automatisch ein **X.509-Zertifikat** aus (SPIFFE-ID)
- Format: `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`
- Zertifikate werden über die **Istio CA (istiod)** ausgestellt und regelmäßig rotiert
- Der Traffic zwischen Sidecars (Classic Mode) bzw. ztunnel (Ambient Mode) wird transparent verschlüsselt

#### Modi

| Modus | Verhalten |
|-------|-----------|
| `PERMISSIVE` | Akzeptiert mTLS **und** Plaintext (Default, gut für Migration) |
| `STRICT` | **Nur** mTLS erlaubt — Plaintext wird abgelehnt |
| `DISABLE` | mTLS deaktiviert |
| `UNSET` | Erbt Einstellung vom Parent (Mesh → Namespace → Workload) |

#### Beispiel: Mesh-weit STRICT

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

#### Beispiel: Namespace-Level

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

#### Beispiel: Port-Level Exception

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: my-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-service
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: PERMISSIVE  # Health-Check-Port ohne mTLS
```

#### Vererbungshierarchie

```
Mesh-Policy (istio-system)
  └── Namespace-Policy
        └── Workload-Policy (mit selector)
              └── Port-Level Override
```

Die spezifischste Policy gewinnt.

---

### 2. Zugriffskontrolle mit AuthorizationPolicy

#### Warum reicht mTLS nicht?

mTLS stellt sicher, **wer** ein Service ist (Identität) und verschlüsselt den Traffic. Aber ohne AuthorizationPolicy kann **jeder** authentifizierte Service **jeden** anderen erreichen. Das ist kein Zero Trust.

#### Aktionstypen

| Action | Verhalten |
|--------|-----------|
| `ALLOW` | Erlaubt Zugriff wenn Regeln matchen |
| `DENY` | Verweigert Zugriff wenn Regeln matchen (hat Vorrang vor ALLOW) |
| `CUSTOM` | Delegiert Entscheidung an externen Authorizer |

#### Evaluierungsreihenfolge

```
1. CUSTOM → 2. DENY → 3. ALLOW → 4. Implizites Verhalten
```

Wenn **keine** AuthorizationPolicy existiert → alles erlaubt.  
Wenn **nur** ALLOW-Policies existieren → alles andere wird implizit verweigert.

#### Beispiel: Nur frontend darf auf backend zugreifen

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

#### Beispiel: Deny-Rule für sensible Endpoints

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-admin
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: DENY
  rules:
  - to:
    - operation:
        paths: ["/admin*"]
    from:
    - source:
        notPrincipals: ["cluster.local/ns/production/sa/admin-service"]
```

#### Wichtige Source-Felder (L4)

| Feld | Beschreibung |
|------|-------------|
| `principals` | SPIFFE-ID des aufrufenden Service |
| `namespaces` | Source-Namespace |
| `ipBlocks` | Source-IP-Bereiche |

#### Wichtige Operation-Felder (L7)

| Feld | Beschreibung |
|------|-------------|
| `methods` | HTTP-Methoden (GET, POST, ...) |
| `paths` | URL-Pfade |
| `hosts` | Ziel-Hostname |
| `ports` | Ziel-Ports |

> **Hinweis:** L7-Felder (methods, paths, hosts) erfordern im Ambient Mode einen **Waypoint Proxy**. Der ztunnel arbeitet nur auf L4.

---

### 3. End-User-Authentifizierung mit RequestAuthentication

#### Wofür?

PeerAuthentication sichert **Service-zu-Service**. RequestAuthentication validiert **End-User-Tokens** (JWTs), die von einem externen Identity Provider stammen (z.B. Keycloak, Auth0, Dex).

#### Beispiel: JWT-Validierung

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  jwtRules:
  - issuer: "https://keycloak.example.com/realms/myrealm"
    jwksUri: "https://keycloak.example.com/realms/myrealm/protocol/openid-connect/certs"
    forwardOriginalToken: true
```

#### Verhalten

- Request **ohne** Token → wird **durchgelassen** (RequestAuthentication prüft nur vorhandene Tokens!)
- Request **mit gültigem** Token → wird durchgelassen, Claims werden extrahiert
- Request **mit ungültigem** Token → wird abgelehnt (401)

#### Kombination mit AuthorizationPolicy

Um Requests **ohne** Token abzulehnen, braucht man eine ergänzende AuthorizationPolicy:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["https://keycloak.example.com/realms/myrealm/*"]
    when:
    - key: request.auth.claims[groups]
      values: ["admin", "editor"]
```

> **Wichtig:** `RequestAuthentication` allein erzwingt **keine** Authentifizierung. Erst zusammen mit einer `AuthorizationPolicy` auf `requestPrincipals` wird ein Token tatsächlich **erforderlich**.

---

### Zusammenspiel der drei Säulen

```
                     Eingehender Request
                            │
                ┌───────────┴───────────┐
                │  PeerAuthentication   │
                │  (mTLS-Identität)     │
                └───────────┬───────────┘
                            │
                ┌───────────┴───────────┐
                │ RequestAuthentication │
                │ (JWT-Validierung)     │
                └───────────┬───────────┘
                            │
                ┌───────────┴───────────┐
                │  AuthorizationPolicy  │
                │  (Zugriffskontrolle)  │
                └───────────┬───────────┘
                            │
                      ┌─────┴─────┐
                      │  Service  │
                      └───────────┘
```

#### Typische Konfigurationsreihenfolge

1. **PeerAuthentication** auf `STRICT` setzen (Mesh-weit oder Namespace)
2. **AuthorizationPolicy** erstellen (wer darf wohin)
3. **RequestAuthentication** hinzufügen (wenn End-User-Tokens validiert werden sollen)
4. **AuthorizationPolicy** mit `requestPrincipals` ergänzen (Token erzwingen + Claims prüfen)

---

### Debugging & Verifizierung

#### mTLS-Status prüfen

```bash
## Prüfen ob mTLS aktiv ist
istioctl x describe pod <pod-name> -n <namespace>

## PeerAuthentication-Policies anzeigen
kubectl get peerauthentication --all-namespaces

## TLS-Verbindung verifizieren
istioctl proxy-config secret <pod-name> -n <namespace>
```

#### AuthorizationPolicy testen

```bash
## RBAC-Debug-Logs aktivieren
istioctl proxy-config log <pod-name> --level rbac:debug

## Logs prüfen
kubectl logs <pod-name> -c istio-proxy | grep rbac

## Typische Log-Einträge:
## "enforced denied" → Policy hat geblockt
## "enforced allowed" → Policy hat erlaubt
```

#### JWT-Probleme debuggen

```bash
## Token manuell prüfen
kubectl exec <pod-name> -c istio-proxy -- \
  curl -s http://localhost:15000/config_dump | \
  jq '.configs[] | select(.["@type"] | contains("listener"))'

## JWT-Rejection im Access-Log
## Response-Flag "UAEX" = Unauthorized External (JWT ungültig)
```

---

### Classic Mode vs. Ambient Mode

| Aspekt | Classic (Sidecar) | Ambient (ztunnel + Waypoint) |
|--------|-------------------|------------------------------|
| mTLS | Sidecar-zu-Sidecar | ztunnel-zu-ztunnel |
| L4-Policy | Im Sidecar | Im ztunnel |
| L7-Policy | Im Sidecar | **Nur mit Waypoint Proxy** |
| RequestAuthentication | Im Sidecar | **Nur mit Waypoint Proxy** |
| Zertifikate | Pro Pod | Pro Node (ztunnel) / Pro SA (Waypoint) |

> **Praxis-Tipp:** Im Ambient Mode zunächst nur L4-Policies (principals, namespaces) nutzen. Für L7 (paths, methods, JWT) einen Waypoint deployen.

### Übung Säule 1: mTLS prüfen und aktivieren/deaktivieren


### Ziel

mTLS-Status im Mesh inspizieren, PeerAuthentication-Policies anwenden und die Auswirkungen auf den Traffic anhand der Bookinfo-Anwendung verifizieren.

---

### Voraussetzungen

- Kubernetes-Cluster mit Istio (Sidecar- oder Ambient-Mode)
- `istioctl` installiert
- Namespace `bookinfo` mit Istio-Injection:

```bash
kubectl create ns bookinfo
kubectl label ns bookinfo istio-injection=enabled
```

---

### Schritt 1: Bookinfo deployen

   * Bereit deployed

**Architektur-Überblick:**

```
productpage → details
            → reviews-v1 (keine Sterne)
            → reviews-v2 → ratings (schwarze Sterne)
            → reviews-v3 → ratings (rote Sterne)
```

#### Funktionstest

```bash
kubectl -n bookinfo exec deploy/productpage-v1 -- \
  curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

Erwartete Ausgabe: `<title>Simple Bookstore App</title>`

---

### Schritt 2: Aktuellen mTLS-Status prüfen

#### 2a) Mit istioctl

```bash
## mTLS-Status der productpage prüfen
istioctl x describe pod \
  $(kubectl -n bookinfo get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') \
  -n bookinfo

## mTLS-Status von ratings prüfen
istioctl x describe pod \
  $(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') \
  -n bookinfo
```

**Frage:** Welchen mTLS-Mode zeigt Istio standardmäßig an?

#### 2b) Bestehende PeerAuthentication-Policies anzeigen

```bash
## Mesh-weit (istio-system)
kubectl get peerauthentication -A

## Im Namespace
kubectl get peerauthentication -n bookinfo
```

> **Hinweis:** PeerAuthentication ist eine **Namespaced Resource** (`NAMESPACED=true`).
> Die Geltung hängt davon ab, *wo* sie erstellt wird:
>
> - Im **App-Namespace** (z.B. `bookinfo`) → gilt nur für diesen Namespace
> - In **`istio-system`** → wirkt als Mesh-weite Policy (quasi cluster-weit)
>
> Es gibt keine echte Cluster-Resource – die „globale" Wirkung entsteht nur durch die Konvention, dass Istio eine PeerAuthentication in `istio-system` als Mesh-Default behandelt.
>
> ```bash
> kubectl api-resources | grep peerauthentication
> # peerauthentications   pa   security.istio.io/v1   true   PeerAuthentication
> ```
>
> **Wichtig:** Wenn weder im App-Namespace noch in `istio-system` eine PeerAuthentication gesetzt ist, zeigt `kubectl get peerauthentication -A` ein leeres Ergebnis (`No resources found`). In diesem Fall greift automatisch **PERMISSIVE** – dieser Default ist im Istio-Quellcode hardcoded und nicht über die meshConfig konfigurierbar. Prüfen lässt sich der effektive Mode mit:
>
> ```bash
> istioctl x describe pod <pod-name> -n bookinfo
> # Ausgabe enthält: "mTLS mode: PERMISSIVE"
> ```

#### 2c) Verifizierung über Request-Header

```bash
## sleep als Debug-Client deployen
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml
kubectl -n bookinfo wait --for=condition=ready pod -l app=sleep --timeout=60s

## Header bei ratings prüfen – XFCC-Header zeigt mTLS-Identität
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -v http://ratings:9080/ratings/0 2>&1 | grep -i x-forwarded-client-cert
```

**Frage:** Welche `X-Forwarded-Client-Cert`-Header sehen Sie? Was bedeutet das?

---

### Schritt 3: mTLS auf STRICT setzen

#### 3a) Namespace-weite STRICT Policy

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```

#### 3b) Verifizieren – Kommunikation im Mesh funktioniert weiterhin

```bash
## Falls nicht vorher schon erstellt 
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml

## productpage kann weiterhin details und reviews erreichen
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage:9080/productpage

## sleep kann ratings erreichen
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings:9080/ratings/0
```

#### 3c) Zugriff ohne Sidecar testen

Starten Sie einen Pod **ohne** Sidecar und versuchen Sie, die Bookinfo-Services zu erreichen:

```bash
## Pod ohne Sidecar in einem Namespace ohne Injection
kubectl create ns no-mesh
kubectl -n no-mesh apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml
kubectl -n no-mesh wait --for=condition=ready pod --all --timeout=60s

## Zugriff auf productpage versuchen
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage

## Zugriff auf ratings versuchen
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0
```

**Erwartetes Ergebnis:** Die Verbindungen schlagen fehl, da der Client kein mTLS-Zertifikat präsentiert. curl gibt den HTTP-Statuscode 000 aus (keine HTTP-Antwort) und bricht mit Exit Code 56 (CURLE_RECV_ERROR) ab – Envoy resettet die Verbindung bereits auf TLS-Ebene, bevor es zur HTTP-Kommunikation kommt.

---


### Schritt 4: mTLS auf PERMISSIVE setzen

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: PERMISSIVE
EOF
```

#### Verifizieren

```bash
## Aus dem Mesh – funktioniert (mit mTLS)
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage:9080/productpage

## Von außerhalb des Mesh – funktioniert jetzt auch (ohne mTLS)
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage
```

**Frage:** Warum ist PERMISSIVE der Istio-Default und wann ist STRICT besser?

---

###

### Schritt 5: Workload-spezifisches mTLS

mTLS lässt sich gezielt für einzelne Services konfigurieren. Hier setzen wir nur den ratings-Service auf STRICT, während der Rest PERMISSIVE bleibt:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: ratings-strict
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  mtls:
    mode: STRICT
EOF
```

#### Verifizieren

```bash
## ratings ist jetzt STRICT – von außerhalb nicht erreichbar
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings.bookinfo:9080/ratings/0

## productpage ist weiterhin PERMISSIVE – von außerhalb erreichbar
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage

## reviews (im Mesh) kann ratings weiterhin erreichen
kubectl -n bookinfo exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" ratings:9080/ratings/0
```

**Frage:** In welcher Reihenfolge wertet Istio PeerAuthentication-Policies aus?

```bash
## Aufräumen
kubectl -n bookinfo delete peerauthentication ratings-strict
```

---

### Schritt 6: Port-Level mTLS (Bonus)

mTLS lässt sich auch pro Port konfigurieren:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: ratings-per-port
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  mtls:
    mode: STRICT
  portLevelMtls:
    9080:
      mode: PERMISSIVE
EOF
```

**Frage:** Für welches Szenario ist Port-Level-Konfiguration sinnvoll?

```bash
kubectl -n bookinfo delete peerauthentication ratings-per-port
```

---

### Schritt 7: mTLS mit DestinationRule steuern

Die **Client-Seite** wird über DestinationRules konfiguriert:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ratings-mtls
  namespace: bookinfo
spec:
  host: ratings.bookinfo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

**Hinweis:** Seit Istio 1.x sendet der Sidecar automatisch mTLS, wenn das Ziel einen Sidecar hat (Auto-mTLS). Eine explizite DestinationRule ist nur nötig, wenn man Auto-mTLS überschreiben will.

```bash
kubectl -n bookinfo delete destinationrule ratings-mtls
```

---

### Schritt 8: mTLS-Traffic mit tcpdump verifizieren (Bonus)

**Terminal 1** – Listener auf dem ratings-Pod:

```bash
kubectl -n bookinfo debug -it \
  $(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') \
  --image=nicolaka/netshoot -- tcpdump -i eth0 -A -c 20 port 9080
```

**Terminal 2** – Request auslösen:

```bash
kubectl -n bookinfo exec deploy/sleep -- curl -s ratings:9080/ratings/0
```

**Frage:** Sehen Sie Klartext oder verschlüsselte Daten im tcpdump-Output?

---

### Aufräumen

```bash
kubectl -n bookinfo delete pod sleep
kubectl -n bookinfo delete sa sleep 
kubectl delete ns no-mesh
```

---

### Zusammenfassung

| Konzept | Beschreibung |
|---------|-------------|
| **PeerAuthentication** | Server-Seite: akzeptiert mTLS (STRICT) oder beides (PERMISSIVE) |
| **DestinationRule** | Client-Seite: sendet mTLS (ISTIO_MUTUAL) oder Klartext (DISABLE) |
| **Auto-mTLS** | Istio aktiviert mTLS automatisch, wenn beide Seiten im Mesh sind |
| **PERMISSIVE** | Default – erlaubt Migration ohne Downtime |
| **STRICT** | Zero-Trust – nur mTLS-Traffic erlaubt |
| **Policy-Hierarchie** | Mesh-weit → Namespace → Workload (spezifischste gewinnt) |

---

### Lösungen

<details>
<summary>Antworten zu den Fragen</summary>

**Schritt 2a:** Istio zeigt standardmäßig `PERMISSIVE` an – Plaintext und mTLS werden akzeptiert.

**Schritt 2c:** Der Header `X-Forwarded-Client-Cert` enthält die SPIFFE-ID des Callers (z.B. `spiffe://cluster.local/ns/bookinfo/sa/sleep`). Das beweist, dass mTLS aktiv ist.

**Schritt 4:** PERMISSIVE ist der Default, weil es eine schrittweise Migration erlaubt – Services können nach und nach ins Mesh aufgenommen werden, ohne dass bestehende Verbindungen brechen. STRICT ist besser für Produktionsumgebungen mit Zero-Trust-Anforderungen.

**Schritt 5:** Istio wertet PeerAuthentication in dieser Reihenfolge aus: Workload-spezifisch (mit `selector`) > Namespace-weit (ohne `selector`, im App-Namespace) > Mesh-weit (ohne `selector`, in `istio-system`). Die spezifischste Policy gewinnt.

**Schritt 6:** Port-Level-Konfiguration ist sinnvoll, wenn ein Service Health-Check-Endpoints hat, die von außerhalb des Mesh (z.B. von einem Load Balancer ohne Sidecar) erreichbar sein müssen.

**Schritt 8:** Bei aktivem mTLS sehen Sie im tcpdump nur verschlüsselte TLS-Daten, keine Klartext-HTTP-Header oder -Bodies.

</details>

### Übung Säule 2 + 3: Authentication und Authorization mit JWT und RBAC


### Grundlagen 

#### Was ist JWKS ? 

**JWKS = JSON Web Key Set**

Ein JSON-Dokument, das die **öffentlichen Schlüssel** enthält, mit denen JWTs (JSON Web Tokens) verifiziert werden können.

**Typisches Format:**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "abc123",
      "use": "sig",
      "n": "0vx7agoebGc...",
      "e": "AQAB"
    }
  ]
}
```

**Wie es funktioniert:**
1. Ein Identity Provider (Keycloak, Auth0, Google etc.) signiert JWTs mit seinem **Private Key**
2. Der JWKS-Endpunkt (z.B. `https://idp.example.com/.well-known/jwks.json`) stellt die zugehörigen **Public Keys** bereit
3. Istio (oder ein anderer Verifier) holt sich die Keys von dort und prüft damit die JWT-Signatur

**In Istio konkret** — `RequestAuthentication`:
```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
spec:
  jwtRules:
  - issuer: "https://idp.example.com"
    jwksUri: "https://idp.example.com/.well-known/jwks.json"
```

Istio cached die Keys und rotiert automatisch mit, wenn der IDP neue Keys veröffentlicht (via `kid` — Key ID).

**Kurz:** JWKS ist der standardisierte Weg, wie ein JWT-Verifier an die Public Keys kommt, ohne sie manuell konfigurieren zu müssen.

#### Begriffe 

**`kid`** (Key ID) — Eindeutige ID des Schlüssels. Damit weiß der Verifier, welcher Key aus dem Set zum JWT passt (das JWT hat `kid` im Header).

**`use`** (Public Key Use) — Wofür der Key gedacht ist:
- `"sig"` = Signaturprüfung (Standard bei JWT)
- `"enc"` = Verschlüsselung

**`n`** (Modulus) — Der RSA-Modulus, Base64url-kodiert. Das ist der mathematische Kern des öffentlichen RSA-Schlüssels.

**`e`** (Exponent) — Der RSA-Exponent, Base64url-kodiert. Fast immer `"AQAB"` (= 65537 dezimal), der Standard-Public-Exponent.

**Zusammenspiel:** `n` und `e` zusammen **sind** der öffentliche RSA-Schlüssel. Damit kann die JWT-Signatur mathematisch verifiziert werden, ohne den Private Key zu kennen.



### Step 0: Preparation 

```
cd
mkdir -p manifests/jwt
cd manifests/jwt 
```

### Step 1: Create http-bin and curl workloads 

```
kubectl create ns foo
kubectl apply -f <(istioctl kube-inject -f ~/istio/samples/httpbin/httpbin.yaml) -n foo
kubectl apply -f <(istioctl kube-inject -f ~/istio/samples/curl/curl.yaml) -n foo
```

### Step 2: Can we connect ? 

```
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
```

### Step 3: Create a RequestAuthentication 

```
nano 01-ra.yml 
```

```
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: "jwt-example"
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/jwks.json"
```

```
kubectl apply -f . 
```

### Step 4: Check with an invalid jwt 

  * Invalid is restricted, so we do not get access (no 200) 

```
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer invalidToken" -w "%{http_code}\n"
```

### Step 5: But: without a jwt -> its work 

  * ... Because ! -> There is no AuthorizationPolicy

```
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
```

### Step 6: We create an AuthorizationPolicy 

>[!NOTE]
>requestPrincipal set to testing@secure.istio.io/testing@secure.istio.io. Istio constructs the requestPrincipal by combining the iss and sub of the JWT token with a / separator.
>
>iss = issuer
>sub = subject 

```
nano 02-ap.yml
```

```
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
```

```
kubectl apply -f 02-ap.yml
```

### Step 7: Test access 

  * jwt consists of 3 parts
    * HEADER / PAYLOAD / SIGNATURE
    * Each part is base64 encoded
  * cut -d. -f2 -> gets the 2nd part -> the payload 
   
```
## This is the way we get the token
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode
```

```
echo $TOKEN
```

```
## Testing with allowed jwt
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
```

```
## Testing without a jwt
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
```

### Step 8: Update AuthorizationPolicy also needing a specific group 

```
nano 02-ap-group.yml
```

```
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group1"]
```

```
kubectl apply -f 02-ap-group.yml
```

### Step 9: get token included a claim for a group 

* Get the JWT that sets the groups claim to a list of strings: group1 and group2:

```
TOKEN_GROUP=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/groups-scope.jwt -s) && echo "$TOKEN_GROUP" | cut -d '.' -f2 - | base64 --decode
```

### Step 10: Test it with that token (so group1 must be included) 

```
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN_GROUP" -w "%{http_code}\n"
```

### Step 11: Test with a token without group included 

  * We use that TOKEN before, which had not group 

```
kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
```

### Step 12: Cleanup 

```
kubectl delete namespace foo
```


### Reference: 

  * https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/
  * JWT Debugger: https://www.jwt.io/

## Sicherheit, Fehlertoleranz & Observability    

### ZeroTrust für bookinfo Namespace bookinfo einrichten


### Ziel

In dieser Übung setzen wir **Zero Trust** im Mesh um:  
**Deny All → explizit erlauben, was nötig ist.**

Jeder Service darf nur mit den Services kommunizieren, die er tatsächlich braucht – und nur über die erlaubten HTTP-Methoden und Pfade.

### Voraussetzungen

- Kubernetes-Cluster mit Istio (Sidecar)
- Bookinfo-App deployed im Namespace `bookinfo`
- mTLS im Mesh aktiv (Default: `STRICT` oder `PERMISSIVE` mit PeerAuthentication)
- Gateway konfiguriert (Gateway API)

### Architektur – Bookinfo Traffic Flow

```
Client → Gateway → productpage → reviews → ratings
                                → details
```

- `productpage` ruft `reviews` und `details` auf
- `reviews` (v2/v3) ruft `ratings` auf
- `details` hat keine Downstream-Dependencies

---


### Schritt 1: GATEWAY_IP setzten

```
## IP notieren 
kubectl -n bookinfo get gateway 
GATEWAY_IP=<ip-des-gateways>
```

### Schritt 2: Projekt-Ordner setzen 

```
cd
mkdir -p manifests/zero-trust-rbac
cd manifests/zero-trust-rbac  
```

### Schritt 3: Ausgangszustand herstellen.

  * Wir wollen das mit Gateway API und httproute testen

```
## just in case 
kubectl -n bookinfo delete virtualservice details productpage reviews
kubectl -n bookinfo delete destinationrule details productpage ratings reviews
```

```
## alles auf gateway ausrichten
kubectl -n bookinfo apply -f  ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo get all 
```


### Schritt 4: mTLS verifizieren

Bevor Authorization Policies greifen, muss mTLS aktiv sein. Ohne mTLS funktionieren `source.principals` und `source.namespaces` nicht.

```bash
## PeerAuthentication prüfen
kubectl get peerauthentication -A
```

```bash
## Falls nicht vorhanden, mesh-weit STRICT setzen
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF
```

#### Verifizieren

```bash
## Aus einem Pod ohne Sidecar sollte der Zugriff fehlschlagen
## weil dieser die Verbindung plaintext und nicht verschlüsselt aufbaut
kubectl run test-no-mesh --image=busybox --rm -it -n default 
```

```
## in der shell
wget -O - http://productpage.bookinfo:9080/productpage
## ** Erwartung: ** Connection reset by peer (kein mTLS-Zertifikat)
```
---

### Schritt 5: Deny All – Alles sperren

Die Basis von Zero Trust: **Erstmal alles verbieten.**

Eine leere `AuthorizationPolicy` ohne Rules mit impliziter `ALLOW`-Action blockiert sämtlichen Traffic im Namespace.

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: bookinfo
spec: {}
EOF
```

#### Verifizieren

```bash
## Über den Browser oder curl auf die Productpage zugreifen
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP/productpage
```

```
##Erwartung: 403
```

> **Erklärung:** Eine Policy mit `spec: {}` (keine `action`, keine `rules`) bedeutet:  
> Action ist implizit `ALLOW`, aber es gibt keine Rules die matchen → kein Request wird erlaubt.

---

### Schritt 6: Gateway → productpage erlauben

Der erste erlaubte Pfad: Das Gateway darf die Productpage erreichen.

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage-from-gateway
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-gateway-istio"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/productpage", "/static/*", "/login", "/logout", "/api/v1/products*"]
EOF
```

> **Hinweis:** Bei Gateway API mit Waypoint Proxy oder anderem Setup muss der `principals`-Wert angepasst werden.  
> Den korrekten Service Account findest du mit:
> ```bash
> kubectl get pods -n istio-system -l app=istio-ingressgateway -o jsonpath='{.items[0].spec.serviceAccountName}'
> ```

#### Verifizieren

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP/productpage
```

```
## Erwartung: 200 OK – aber die Seite zeigt Fehler für Reviews und Details (noch blockiert)
## --> Aber: Es ist 403.
``` 

---

### Schritt 7: Problem rbac debuggen (am productpage - sidecar) 

```
## Kommt die Anfrage am productpage (sidecar- container an) ?
## Dazu müssen wir zunächst wissen, ob rbac:debug gesetzt ist, nur dann sehen wir etwas
istioctl proxy-config log deploy/productpage-v1 -n bookinfo | grep rbac
## Falls nicht rbac: debug,  jetzt setzen:
istioctl proxy-config log deploy/productpage-v1 -n bookinfo --level rbac:debug
```

```
## Anfrage nochmal absetzen und Log überprüfen
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP/productpage
## Es kommt nichts an
kubectl logs -n bookinfo deploy/productpage-v1 -c istio-proxy --tail=30 | grep rbac
```

  * **Resultat**: Da kommt nichts an, das Problem muss also davor sein: AM Gateway 


### Schritt 8: Problem rbac (am gateway - pod) identifizieren 

#### Situation: 

  * Anfrage kommt garnicht bis productpage -> Sidecar
  * Problem muss also vorher bereits existieren
  * Vermutung: Am Gateway

#### Debuggen

```
## Gateway verifizieren
kubectl -n bookinfo get gateway
## Dies erzeugt einen Pod -> Das ist unser Freund 
kubectl -n bookinfo get pods | grep gateway
```

```
## rbac einschalten / Abfrage absenden und nochmal überprüfen 
istioctl proxy-config log deploy/bookinfo-gateway-istio -n bookinfo --level rbac:debug
curl -s http://$GATEWAY_IP/productpage
kubectl logs -n bookinfo deploy/bookinfo-gateway-istio -c istio-proxy --tail=30 | grep rbac
```

![Ausgabe vom Pod](image-1.png)


### Schritt 9: Problem rbac (am gateway - pod) lösen 

   * Anfragen werden nicht durchgelassen, die vom Web-Browser kommen 
   * Diese müssen erlaubt werden 

```
## Hier sehen wir die labels 
kubectl -n bookinfo get pods --show-labels | grep gateway
```

```
nano allow-gateway-inbound.yaml 
```

```
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-gateway-inbound
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      gateway.networking.k8s.io/gateway-name: bookinfo-gateway
  rules:
  - to:
    - operation:
        methods: ["GET"]
        paths: ["/productpage", "/static/*", "/login", "/logout", "/api/v1/products*"]
```

```
kubectl apply -f . 
```

```
## Nochmal testen 
## Gibt es für das gateway jetzt eine Policy 
istioctl x authz check deploy/bookinfo-gateway-istio -n bookinfo 
```

```
## Nochmal aufrufen
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_IP/productpage
```

```
## For fun: wie sieht der RBAC - Eintrag aus 
kubectl logs -n bookinfo deploy/bookinfo-gateway-istio -c istio-proxy --tail=30 | grep rbac
```

 * Ergebnis: Es klappt. Eintrag: allowed by default or by policy 


### Schritt 10: productpage → details erlauben

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-details
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/details/*"]
EOF
```

#### Verifizieren

```bash
## Achtung es kann einen Moment dauern, bis die Rechte im Mesh bekannt sind 
curl -s http://$GATEWAY_IP/productpage | grep -o "Book Details"
## Erwartung: "Book Details" erscheint – Details-Section wird angezeigt
```

---

### Schritt 11: productpage → reviews erlauben

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-reviews
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/reviews/*"]
EOF
```

---

### Schritt 12: reviews → ratings erlauben

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-ratings
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-reviews"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/ratings/*"]
EOF
```

#### Verifizieren

```bash
curl -s http://$GATEWAY_IP/productpage | grep -o "Reviews"
## Erwartung: Ratings werden angezeigt (Sterne bei reviews v2/v3)
```

---

### Schritt 13: Gesamtstatus prüfen

```bash
## Alle Authorization Policies auflisten
kubectl get authorizationpolicies -n bookinfo

## Erwartete Ausgabe:
## NAME                             AGE
## deny-all                         ... 
## allow-gateway-inbound            ...
## allow-productpage-from-gateway   ...
## allow-details                    ...
## allow-reviews                    ...
## allow-ratings                    ...
```

#### Negativtests – Nicht erlaubte Kommunikation

```bash
## Test 1: ratings darf NICHT direkt von productpage aufgerufen werden
```
```bash
## Debug Container verwenden 

POD=$(kubectl get pods -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
kubectl debug $POD -n bookinfo --image=curlimages/curl -it -- sh
```

Dann im Container:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://ratings:9080/ratings/0
## Erwartung: 403
```


```bash
## Test 2: details darf NICHT reviews aufrufen
POD=$(kubectl get pods -n bookinfo -l app=details -o jsonpath='{.items[0].metadata.name}')
kubectl debug $POD -n bookinfo --image=curlimages/curl -it -- sh
```

Dann im Container:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://reviews:9080/reviews/0
## Erwartung: 403
```

```
## Test 3: POST auf productpage ist nicht erlaubt (nur GET)
kubectl apply -f ~/istio/samples/sleep/sleep.yaml -n bookinfo
kubectl exec deploy/sleep -n bookinfo -c sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://productpage:9080/productpage
## Erwartung: 403
```

---

### Schritt 14: DENY-Policy – Explizit blockieren

DENY-Policies werden **vor** ALLOW evaluiert und eignen sich für zusätzliche Einschränkungen.

Beispiel: Zugriff auf `/ratings` mit bestimmtem Header blockieren:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-ratings-debug
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  action: DENY
  rules:
  - when:
    - key: request.headers[x-debug]
      values: ["true"]
EOF
```

#### Verifizieren

```bash
POD=$(kubectl get pods -n bookinfo -l app=reviews -o jsonpath='{.items[0].metadata.name}')
kubectl debug $POD -n bookinfo --image=curlimages/curl -it -- sh
```

Dann im Container:

```bash
## Normaler Zugriff - Erwartung: 200
curl -s -o /dev/null -w "%{http_code}\n" http://ratings:9080/ratings/0

## Mit x-debug Header - Erwartung: 403
curl -s -o /dev/null -w "%{http_code}\n" -H "x-debug: true" http://ratings:9080/ratings/0
```

---

### Schritt 15: Alternative Namespace-Isolation (Bonus) - statt 

Services aus anderen Namespaces komplett blockieren:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-other-namespaces
  namespace: bookinfo
spec:
  action: DENY
  rules:
  - from:
    - source:
        notNamespaces: ["bookinfo", "istio-system"]
EOF
```

---

### Zusammenfassung – Zero Trust Checkliste

| Prinzip | Umsetzung |
|---|---|
| **mTLS erzwingen** | `PeerAuthentication` mit `mode: STRICT` |
| **Default Deny** | Leere `AuthorizationPolicy` im Namespace |
| **Least Privilege** | Pro Service nur erlaubte Quellen, Methoden und Pfade |
| **Defense in Depth** | DENY-Policies für zusätzliche Einschränkungen |
| **Namespace-Isolation** | `notNamespaces` um Cross-Namespace-Zugriff zu blockieren |

#### Evaluierungsreihenfolge

```
1. CUSTOM Policies  (externe Authorizer)
2. DENY Policies    (blockt wenn match)
3. ALLOW Policies   (erlaubt wenn match)
4. Keine Policy     → Default: ALLOW
5. Leere Policy     → Default: DENY (kein match möglich)
```

---

### Aufräumen

```bash

kubectl delete authorizationpolicy -n bookinfo --all
## When no peerauthentication is set 
## it will be PERMISSIVE by default 
kubectl delete peerauthentication default -n istio-system
```

---

### Troubleshooting

```bash
## Envoy-Logs auf RBAC-Entscheidungen prüfen
kubectl logs deploy/productpage-v1 -n bookinfo -c istio-proxy | grep "rbac"

## Debug-Level für RBAC setzen
istioctl proxy-config log deploy/productpage-v1 -n bookinfo --level rbac:debug

## Effektive Policies für einen Pod anzeigen
istioctl x authz check deploy/productpage-v1 -n bookinfo
```

## Demo-App (istio api) 

### Istio demo-app *bookinfo* installieren (istio API - Variante)


### Überblick 

<img width="992" height="615" alt="image" src="https://github.com/user-attachments/assets/5773ce04-fd83-45a6-9914-d2b1b72c1505" />

### Vorbereitung

#### Namespace erstellen und labeln. 
```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

### bookdemo app ausrollen 


```
kubectl -n bookinfo apply -f  ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo get all 
```

### testen ob die app funktioniert 

```
kubectl -n bookinfo exec "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

### App mit istio-ingress-gateway nach aussen öffnen 

```
cd
mkdir -p manifests/istio-gateway
cd manifests/istio-gateway
nano gateway-virtualservice.yaml
```

```
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingress # use istio default controller
  servers:
  - port:
      number: 80
      name: http2
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

```
kubectl -n bookinfo apply -f .
```



```
kubectl -n bookinfo get gw
kubectl -n bookinfo get virtualservice -o yaml 
```

```
## get ip from here
kubectl -n istio-ingress get all
kubectl -n istio-ingress get svc 
```

```
http://<external-ip>/productpage 
## or in your browser
```



## Übungen Steuerung des Netzwerkverhaltens (istio api)

### Übung: Header-basiertes Routing - istio api

## Request Routing 

### Vorher (ohne request routing) 

  * Es werden alle Pods angezeigt, die das Label: app:reviews haben
  * D.h. jedesmal wenn ich die Seite öffne, wird eine andere Version angegezeigt (v1, v2 oder v3)
  * Service (selector: app:reviews)

```
kubectl -n bookinfo get svc reviews -o yaml
kubectl -n bookinfo get pods -l app=reviews --show-labels
```

### Übung (jetzt request - routing) 

**Voraussetzung:**

- Bookinfo-App läuft bereits im Namespace `bookinfo` (ansonsten neu einrichten)
- Service Reviews ist definiert
- Es gibt 3 verschieden Pods an Reviews (v1, v2 und v3)
- Ingress/Gateway + `GATEWAY_URL` (IP: http://164.90.237.35/productpage aus der vorherigen Übung vorhanden

#### 0 Vorbereitung

```bash
mkdir -p ~/manifests/requests
cd ~/manifests/requests

## Die Destinationen-Versionen anlegen
cp -a ~/istio/samples/bookinfo/networking/destination-rule-all.yaml destination-rule-all.yaml
kubectl -n bookinfo apply -f destination-rule-all.yaml 
```




---

#### 1. VirtualService: Alle Requests → `reviews-v1`

```
nano reviews-v1.yaml 
```

```bash
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

```
kubectl -n bookinfo apply -f reviews-v1.yaml 
kubectl -n bookinfo get virtualservice reviews -n bookinfo
```

```
## Gateway IP ausfinden machen
kubectl -n istio-ingress get svc istio-ingress
GATEWAY_URL=<ip-aus-letzter-ausgabe>

```
## Anzeige im Browser - es ist immer die v1
http://$GATEWAY_URL/productpage
```


---

#### 2. Adjust VirtualHost: User `jason` → `reviews-v2`, Rest → `reviews-v1`

```
cp -a ~/istio/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml virtual-service-reviews-test-v2.yaml
cat virtual-service-reviews-test-v2.yaml
```

```
kubectl -n bookinfo apply -f virtual-service-reviews-test-v2.yaml
kubectl -n bookinfo get vs reviews -n bookinfo -o yaml
```

---

### 3. Testen im Browser

```bash
echo "$GATEWAY_URL"
## Beispiel: http://<IP>:<PORT>

## 1. Im Browser: $GATEWAY_URL/productpage aufrufen (nicht eingeloggt oder anderer User)
##    → Reviews ohne Sterne (v1)

## 2. Im Browser: als User "jason" einloggen
##    → Reviews mit Sternen (v2)
```

---

### 4. Aufräumen

```bash
kubectl -n bookinfo delete vs reviews 
```

### Reference: 

  * https://istio.io/latest/docs/examples/bookinfo/#define-the-service-versions

## Monitoring, Logging & Observability

### Installation Jaeger


### Prerequisites 

  * *.tlnx.do.t3isp.de subdomain is already set up and pointing to your cluster
>[!NOTE]
>Bitte zunächst Prometheus installieren 

### Step 1: Install addon jaeger 

```
cd 
mkdir -p manifests/jaeger
cd manifests/jaeger
cp -a ~/istio/samples/addons/jaeger.yaml .
kubectl apply -f .
kubectl -n istio-system get pods
kubectl -n istio-system get svc
```

### Step 2: Setup basic auth 

```
nano jaeger-basic-auth.yaml
```

```
## Secret for basic auth: user "training" / password "myS3cr3t!"
## Already base64-encoded in htpasswd format.
apiVersion: v1
kind: Secret
metadata:
  name: jaeger-basic-auth
  namespace: istio-system
type: Opaque
data:
  # htpasswd-style content:
  # training:$2b$12$CfOZaJ.Tr0zu6PfpbuCjzeKiQ2PzZARfP.CbC6tRU/7OvEHCIOREm
  auth: dHJhaW5pbmc6JDJiJDEyJENmT1phSi5UcjB6dTZQZnBidUNqemVLaVEyUHpaQVJmUC5DYkM2dFJVLzdPdkVIQ0lPUkVtCg==
```

```
kubectl apply -f .
```

#### Step 3: Setup Ingress 

```
nano ingress.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: jaeger-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - jaeger'
spec:
  ingressClassName: "nginx"
  rules:
    - host: jaeger.tlnXX.app.do.t3isp.de
      # ⬆️ Each trainee replaces "XX" with their number, e.g. jaeger.tln10.do.t3isp.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: tracing
                port:
                  number: 80
```

```
kubectl apply -f .
```

```
## Im browser aufrufen und credentials eingeben (s.o.)
http://jaeger.tlnXX.app.do.t3isp.de
```

### Distributed Tracing mit Jaeger


### Prerequisites 

  * Jaeger is setup
  * demo profile is set up
  * bookinfo is rolled out

### Special when using ambient 

  * because installing with helm does not use demo-profile 

```
cd 
mkdir -p manifests/jaeger
cd manifests/jaeger
nano values.yaml 
```

```
meshConfig:
  accessLogFile: /dev/stdout
  extensionProviders:
    - name: otel
      envoyOtelAls:
        service: opentelemetry-collector.observability.svc.cluster.local
        port: 4317
    - name: skywalking
      skywalking:
        service: tracing.istio-system.svc.cluster.local
        port: 11800
    - name: otel-tracing
      opentelemetry:
        port: 4317
        service: opentelemetry-collector.observability.svc.cluster.local
    - name: jaeger
      opentelemetry:
        port: 4317
        service: jaeger-collector.istio-system.svc.cluster.local        


pilot:
  traceSampling: 100
```

```
kubectl delete validatingwebhookconfigurations istio-validator-istio-system
helm upgrade istiod istio/istiod -n istio-system --set profile=ambient --version 1.29.1 -f values.yaml

```





### Note: 

  * You do NOT need to enable tracing, because it is already active in demo profile
  * You need so set up a telemetry - object

### Walktrough 

```
cd
mkdir -p manifests/tracing-jaeger
cd manifests/tracing-jaeger
```

```
nano 01-telemetry-jaeger.yaml 
```

```
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
```

```
kubectl apply -f .
```

### Test it: 

```
## ADDRESS -> ip
kubectl -n bookinfo get gateway bookinfo-gateway
```

```
## Adjust to your own IP 
GATEWAY_URL=164.90.237.35
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
```

### Open jaeger.tlnxx.do.t3isp.de 

```
Search ->
```

<img width="436" height="147" alt="image" src="https://github.com/user-attachments/assets/0bb5069b-ee3c-4799-91fa-35b498a3c047" />

<img width="444" height="61" alt="image" src="https://github.com/user-attachments/assets/97451972-7070-46ab-a4e4-39dc144f60b1" />


### Cleanup 

```
kubectl delete -f .
```



### Reference 

  * https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/

### Installation Prometheus Addon with Ingress


### Prerequisites 

  * *.tlnx.do.t3isp.de subdomain is already set up and pointing to your cluster
    
### Step 1: Install addon prometheus 

```
cd 
mkdir -p manifests/prometheus
cd manifests/prometheus
cp -a ~/istio/samples/addons/prometheus.yaml .
kubectl apply -f .
kubectl -n istio-system get pods
kubectl -n istio-system get svc
```

### Step 2: Setup basic auth 

```
nano prometheus-basic-auth.yaml
```

```
## Secret for basic auth: user "training" / password "myS3cr3t!"
## Already base64-encoded in htpasswd format.
apiVersion: v1
kind: Secret
metadata:
  name: prometheus-basic-auth
  namespace: istio-system
type: Opaque
data:
  # htpasswd-style content:
  # training:$2b$12$CfOZaJ.Tr0zu6PfpbuCjzeKiQ2PzZARfP.CbC6tRU/7OvEHCIOREm
  auth: dHJhaW5pbmc6JDJiJDEyJENmT1phSi5UcjB6dTZQZnBidUNqemVLaVEyUHpaQVJmUC5DYkM2dFJVLzdPdkVIQ0lPUkVtCg==
```

```
kubectl apply -f .
```

#### Step 3: Setup Ingress 

```
nano ingress.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: prometheus-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - prometheus'
    nginx.ingress.kubernetes.io/hsts: "false"
    nginx.ingress.kubernetes.io/hsts-max-age: "0"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "false"
    nginx.ingress.kubernetes.io/hsts-preload: "false"

spec:
  ingressClassName: "nginx"
  rules:
    - host: prometheus.tlnXX.app.do.t3isp.de
      # ⬆️ Each trainee replaces "XX" with their number, e.g. kiali.tln10.do.t3isp.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus
                port:
                  number: 9090
```

```
kubectl apply -f .
```

```
## Im browser aufrufen und credentials eingeben (s.o.)
http://prometheus.tlnXX.do.t3isp.de
```

### Metriken mit Prometheus auswerten


### Walkthrough 

#### Commandline: Send Traffic to the mesh 

```
## Change to your ip accordingly 
GATEWAY_URL=164.90.237.35
curl "http://$GATEWAY_URL/productpage"
```

#### Search for metrics 

```
## Login to prometheus.tlnxx.do.t3isp.de
## Click on Query
```

<img width="1894" height="79" alt="image" src="https://github.com/user-attachments/assets/6c1dc59a-0d68-4a0c-9081-0f68a692c4bc" />

```
## in enter expression -> enter
istio_requests_total
## THEN -> Click Execute
```

** This will show all the requests to istio **

```
## You can also click on Graph to see the graphical representation
```

<img width="815" height="416" alt="image" src="https://github.com/user-attachments/assets/2cd43b98-4069-4340-8d91-1199c041bb0c" />

#### Try other queries 

```
## 1. Only request to the productpage in namespace bookinfo 
istio_requests_total{destination_service="productpage.bookinfo.svc.cluster.local"}
```

   * Hinweis, wenn du mehrere Einträge siehst wie **hier:**

<img width="1789" height="467" alt="image" src="https://github.com/user-attachments/assets/49881c0f-b462-4ff0-98e3-8bebc22a6566" />

   * .. unterscheiden sich diese bspw. an dieser Stelle durch den Response - Code (200, 403 etc.)

```
## 2. Total count of all requests to v3 of the reviews service:
istio_requests_total{destination_service="reviews.bookinfo.svc.cluster.local", destination_version="v3"}
```

```
## 3.  This query returns the current total count of all requests to the v3 of the reviews service.
## Rate of requests over the past 5 minutes to all instances of the productpage service:
## =~ <- **ist ein regulärer Ausdruck** 
rate(istio_requests_total{destination_service=~"productpage.*", response_code="200"}[5m])
```

### Reference: 

  * https://istio.io/latest/docs/tasks/observability/metrics/querying-metrics/

### Installation Grafana Addon with Ingress


### Prerequisites 

  * *.tlnx.do.t3isp.de subdomain is already set up and pointing to your cluster
>[!NOTE]
>Bitte zunächst Prometheus installieren 

### Step 1: Install addon grafana 

```
cd 
mkdir -p manifests/grafana
cd manifests/grafana
cp -a ~/istio/samples/addons/grafana.yaml .
kubectl apply -f .
kubectl -n istio-system get pods
kubectl -n istio-system get svc
```

### Step 2: Setup basic auth 

```
nano grafana-basic-auth.yaml
```

```
## Secret for basic auth: user "training" / password "myS3cr3t!"
## Already base64-encoded in htpasswd format.
apiVersion: v1
kind: Secret
metadata:
  name: grafana-basic-auth
  namespace: istio-system
type: Opaque
data:
  # htpasswd-style content:
  # training:$2b$12$CfOZaJ.Tr0zu6PfpbuCjzeKiQ2PzZARfP.CbC6tRU/7OvEHCIOREm
  auth: dHJhaW5pbmc6JDJiJDEyJENmT1phSi5UcjB6dTZQZnBidUNqemVLaVEyUHpaQVJmUC5DYkM2dFJVLzdPdkVIQ0lPUkVtCg==
```

```
kubectl apply -f .
```

#### Step 3: Setup Ingress 

```
nano ingress.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: grafana-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - grafana'
    nginx.ingress.kubernetes.io/hsts: "false"
    nginx.ingress.kubernetes.io/hsts-max-age: "0"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "false"
    nginx.ingress.kubernetes.io/hsts-preload: "false"
spec:
  ingressClassName: "nginx"
  rules:
    - host: grafana.tlnXX.app.do.t3isp.de
      # ⬆️ Each trainee replaces "XX" with their number, e.g. grafana.tln10.do.t3isp.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
```

```
kubectl apply -f .
```

```
## Im browser aufrufen und credentials eingeben (s.o.)
http://grafana.tlnXX.app.do.t3isp.de
```

### Grafana Dashboards für istio


### Prerequisites 

  * bookinfo app is installed
  * Grafana dashboard is setup here: http://grafana.tlnxx.do.t3isp.de
  * demo profile is used

### Walkthrough 

```
## Open dashboard
http://grafana.tlnxx.do.t3isp.de
```

```
Go to dashboards
```

<img width="337" height="196" alt="image" src="https://github.com/user-attachments/assets/3d60b6ad-0992-4aed-8324-c21d1e8556e2" />

```
Select Dashboard from folder istio ->
```

<img width="526" height="122" alt="image" src="https://github.com/user-attachments/assets/7337c5b4-cbcb-4a91-9971-a73f627b13b5" />

```
## Send traffic to the /productpage
## commandline
## Adjust to your IP 
GATEWAY_URL=164.90.237.35
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
```

```
## In the dashboard click reload (right corner) - a couple of times 
```

```
<img width="334" height="288" alt="image" src="https://github.com/user-attachments/assets/6d117df1-679a-46c5-b52a-82afe68ce138" />
```

```
## The traffic volumen should increase
```

<img width="388" height="255" alt="image" src="https://github.com/user-attachments/assets/254cb10a-1412-4e58-a55d-8612ea8decf6" />

### Next stop: Service Dashboard

  * Click on Dashboards -> Istio -> Service Dashboard

### Next stop: Workload Dashboad 

  * Click on Dashboard -> Istio -> Workload Dashboard



### Sinnvolle istio Metriken und Dashboards - Teil 1


### Was ist Latenz?

Die Zeit, die ein Request von Absenden bis Antwort braucht — gemessen in Millisekunden. In Istio misst der **Envoy Sidecar** diese Zeit automatisch für jeden Request.

#### Inbound vs. Outbound Latenz

Stell dir Service A → Service B → Service C vor.

**Aus Sicht von Service B:**

- **Inbound-Latenz** = wie lange B insgesamt braucht, um auf einen eingehenden Request zu antworten (gemessen vom Envoy-Proxy *vor* B). Das ist die Gesamtzeit aus Sicht des Aufrufers.
- **Outbound-Latenz** = wie lange die Requests dauern, die B *selbst nach außen* schickt (z.B. an Service C). Gemessen vom Envoy-Proxy *von B* Richtung C.

#### Warum das beim Troubleshooting hilft

```
Inbound-Latenz von B:  800ms
Outbound-Latenz von B → C:  750ms
```

→ B selbst ist nicht das Problem — er wartet fast die ganze Zeit auf C.

```
Inbound-Latenz von B:  800ms
Outbound-Latenz von B → C:  50ms
```

→ B selbst braucht ~750ms für seine eigene Verarbeitung — **B ist der Flaschenhals**.

Die Differenz zwischen Inbound und Outbound zeigt dir also, ob der Service selbst langsam ist oder ob er auf einen anderen wartet.

---

### Goldene Signale in Prometheus

#### Latenz pro Service (P99)

```promql
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
  by (destination_service, le)
)
```

#### Error Rate

```promql
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
by (destination_service)
/
sum(rate(istio_requests_total[5m]))
by (destination_service)
```

#### Throughput / Request Rate

```promql
sum(rate(istio_requests_total[5m])) by (destination_service, source_workload)
```

---

### Grafana Dashboards

Istio bringt fertige Dashboards mit — die wichtigsten für Troubleshooting:

- **Istio Mesh Dashboard** → Überblick über das gesamte Mesh (Global Request Rate, Error Rate)
- **Istio Service Dashboard** → Latenz-Heatmap pro Service, P50/P90/P99 auf einen Blick
- **Istio Workload Dashboard** → Inbound/Outbound Latenz pro Pod, hier siehst du, *welcher* Workload bremst

---

### Systematisch den Engpass eingrenzen

1. **Mesh Dashboard** → welcher Service hat hohe Latenz oder Error Rate?
2. **Service Dashboard** → auf den verdächtigen Service klicken → Inbound vs. Outbound Latenz vergleichen
3. **Workload Dashboard** → einzelne Pods prüfen, ob ein bestimmter Pod langsam ist

**Kernlogik:** Wenn die Inbound-Latenz eines Service hoch ist, aber seine Outbound-Latenz zu Downstream-Services niedrig → der Service selbst ist das Problem. Wenn die Outbound-Latenz hoch ist → das Problem liegt weiter downstream.

---

### Zusätzlich: Kiali + Tracing

- **Kiali** zeigt den Service Graph mit farbcodierten Edges (rot = Fehler, gelb = langsam)
- **Jaeger/Zipkin** für Distributed Tracing → einzelne Requests aufschlüsseln, um zu sehen, *wo genau* im Call-Chain die Zeit verbraucht wird

---

### Nützlicher Alert (PrometheusRule)

```yaml
- alert: HighP99Latency
  expr: |
    histogram_quantile(0.99,
      sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
      by (destination_service, le)
    ) > 500
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "P99 Latenz > 500ms für {{ $labels.destination_service }}"
```

---

### Kurzfassung

Mesh Dashboard → auffälligen Service identifizieren → Service/Workload Dashboard → Inbound vs. Outbound Latenz vergleichen → mit Tracing den konkreten Call pinpointen.

### Sinnvolle istio Metriken und Dashboards - Teil 2

  * https://github.com/jmetzger/training-istio-kubernetes/blob/main/istio/observeability/istio-observeablity-in-prometheus-2.md

### Installation Kiali // Installation


### Prerequisites 

  * *.tlnx.do.t3isp.de subdomain is already set up and pointing to your cluster
>[!NOTE]
>Bitte zunächst Prometheus installieren 

### Step 1: Install addon kiali 

```
cd 
mkdir -p manifests/kiali
cd manifests/kiali
cp -a ~/istio/samples/addons/kiali.yaml .
kubectl apply -f .
kubectl -n istio-system get pods
kubectl -n istio-system get svc
```

### Step 2: Setup basic auth 

```
nano kiali-basic-auth.yaml
```

```
## Secret for basic auth: user "training" / password "myS3cr3t!"
## Already base64-encoded in htpasswd format.
apiVersion: v1
kind: Secret
metadata:
  name: kiali-basic-auth
  namespace: istio-system
type: Opaque
data:
  # htpasswd-style content:
  # training:$2b$12$CfOZaJ.Tr0zu6PfpbuCjzeKiQ2PzZARfP.CbC6tRU/7OvEHCIOREm
  auth: dHJhaW5pbmc6JDJiJDEyJENmT1phSi5UcjB6dTZQZnBidUNqemVLaVEyUHpaQVJmUC5DYkM2dFJVLzdPdkVIQ0lPUkVtCg==
```

```
kubectl apply -f .
```

#### Step 3: Setup Ingress 

```
nano ingress.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kiali
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: kiali-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - kiali'
spec:
  ingressClassName: "nginx"
  rules:
    - host: kiali.tlnXX.do.t3isp.de
      # ⬆️ Each trainee replaces "XX" with their number, e.g. kiali.tlnXX.app.do.t3isp.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kiali
                port:
                  number: 20001
```

```
kubectl apply -f .
```

```
## Im browser aufrufen und credentials eingeben (s.o.)
http://kiali.tlnXX.app.do.t3isp.de
```

### Visualisierung mit Kiali


### Start Traffic on commandline 

```
## Set your ip here, you will get it from
## Look for the line with external ip 
kubectl -n bookinfo get gateway bookinfo-gateway  
GATEWAY_URL=164.90.237.35 

## This consistantly shows 200, without much a-do 
watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage

```

### Open kiali, and see how it goes 

```
in browser:
## Replace xx with your Teilnehmer-Nr. 
http://kiali.tlnxx.do.t3isp.de

## click on Traffic Graph and choose namespace bookinfo
```

### Mesh 

  * Click on mesh to see if everything is functional


### 

### Analyse & Debugging von Service-Mesh-Daten


### Ziel

Einen wiederholbaren Prozess vermitteln, mit dem Teilnehmer Performance-Probleme und Fehler in einem Istio Service Mesh systematisch eingrenzen und beheben können.

### Voraussetzungen

#### Tools im Cluster

| Tool | Zweck | Installation (Demo/Training) |
|---|---|---|
| Prometheus + Grafana | Metrics & Dashboards | `kubectl apply -f samples/addons/prometheus.yaml` / `grafana.yaml` |
| Kiali | Service Graph & Health | `kubectl apply -f samples/addons/kiali.yaml` |
| Jaeger oder Tempo | Distributed Tracing | `kubectl apply -f samples/addons/jaeger.yaml` |

> **Hinweis:** Die Addon-Manifeste aus `samples/addons/` sind nur für Demo/Training geeignet. Für Produktion: eigene Deployments mit Persistence, Retention und HA.

#### Demo-Applikation

Bookinfo-App mit aktiviertem Istio-Sidecar-Injection:

```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

---

### Der Debugging-Workflow

#### Schritt 1: Symptom erkennen (Grafana / Prometheus)

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

#### Schritt 2: Wo im Call-Graph? (Kiali)

Kiali öffnen:

```bash
istioctl dashboard kiali
```

Im Service Graph prüfen:

- Welche Edges zeigen erhöhte Latenz (orange/rot)?
- Ist der betroffene Service selbst langsam, oder sein Downstream?
- Gibt es unerwartete Traffic-Pfade?

**Ergebnis:** Du weißt jetzt, *zwischen welchen Services* das Problem liegt.

#### Schritt 3: Einzelnen Request verfolgen (Jaeger / Tempo)

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

#### Schritt 4: Ursache eingrenzen

| Befund | Mögliche Ursache | Nächster Schritt |
|---|---|---|
| Envoy-Proxy langsam | mTLS-Overhead, Retry-Storms, Rate Limiting | `istioctl proxy-config`, Envoy Stats prüfen |
| App selbst langsam | CPU-Limit, Memory-Pressure, langsame DB-Query | `kubectl top pod`, App-Profiling |
| Downstream langsam | Problem liegt beim Downstream-Service | Workflow dort wiederholen |
| Viele Retries sichtbar | Flapping Backend, falsche Timeout-Config | DestinationRule / VirtualService prüfen |
| Connection-Fehler | mTLS-Mismatch, PeerAuthentication falsch | `istioctl authn tls-check` |

#### Schritt 5: Low-Level Debugging (Envoy / istioctl)

Wenn du tiefer gehen musst:

```bash
## Sync-Status aller Sidecars mit istiod prüfen
istioctl proxy-status

## Cluster-Config eines Pods inspizieren
istioctl proxy-config clusters <pod> -n <ns>

## Routen eines Pods inspizieren
istioctl proxy-config routes <pod> -n <ns>

## Listener eines Pods inspizieren
istioctl proxy-config listeners <pod> -n <ns>

## Envoy-Stats direkt abfragen
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET /stats

## Envoy-Config-Dump (vollständig)
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET /config_dump

## Istio-Analyse auf Fehlkonfigurationen
istioctl analyze -n <ns>
```

---

### Trainings-Lab: "Finde den Fehler"

#### Setup: Latenz per Fault Injection erzeugen

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

#### Aufgabe für Teilnehmer

> "Die Bookinfo-App ist langsam. Findet heraus, welcher Service betroffen ist, wo die Latenz entsteht und was die Ursache ist."

#### Erwarteter Lösungsweg

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

#### Varianten für weitere Labs

| Szenario | Injection | Lernziel |
|---|---|---|
| HTTP-Fehler | `fault.abort: 503, 30%` | Error-Rate Debugging |
| Timeout-Kette | Delay 5s + Timeout 2s in DestinationRule | Timeout-Konfiguration verstehen |
| Circuit Breaker | `outlierDetection` + Last erzeugen | Circuit Breaking analysieren |
| mTLS-Mismatch | `PeerAuthentication: STRICT` + Client ohne Sidecar | mTLS-Fehler debuggen |

---

### Zusammenfassung: Der Workflow auf einen Blick

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

## Skalierung, Erweiterbarkeit & Performance-Optimierung

### Skalierung von Istio - Möglichkeiten/Tipps&Tricks


### Überblick

Istio besteht aus einer **Control Plane** (istiod) und einer **Data Plane** (Envoy-Proxies). Beide müssen unabhängig voneinander skaliert werden. Die größte Herausforderung bei wachsenden Meshes ist nicht CPU oder RAM, sondern die **xDS-Konfigurationsverteilung** – also die Menge an Routing-Regeln, Endpoints und Policies, die istiod an jeden einzelnen Proxy pushen muss.

---

### 1. Control Plane Skalierung (istiod)

#### 1.1 Horizontale Skalierung

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

#### 1.2 HorizontalPodAutoscaler

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

#### 1.3 Ressourcen-Limits

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

### 2. xDS-Konfigurationsumfang reduzieren

Dies ist der **wichtigste Hebel** für Istio-Skalierung. Standardmäßig bekommt jeder Envoy-Proxy die Konfiguration für **alle** Services im Mesh – auch für Services, mit denen er nie kommuniziert.

#### 2.1 Sidecar-Resource

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

#### 2.2 Discovery Selectors (meshConfig)

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
## values-istiod.yaml
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

#### 2.3 exportTo

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

### 3. Data Plane Skalierung

#### 3.1 Sidecar-Ressourcen pro Workload

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

#### 3.2 Envoy Concurrency

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

#### 3.3 Ambient Mode als Skalierungsalternative

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

### 4. Monitoring der Skalierung

#### 4.1 Wichtige istiod-Metriken

| Metrik | Bedeutung | Alarmschwelle |
|--------|-----------|---------------|
| `pilot_xds_pushes` | Anzahl xDS-Pushes | Spikes = häufige Config-Änderungen |
| `pilot_xds_push_time` | Dauer eines Push-Zyklus | > 5s problematisch |
| `pilot_proxy_convergence_time` | Zeit bis alle Proxies aktuell | > 10s kritisch |
| `pilot_virt_services` | Anzahl VirtualServices | Wachstum beobachten |
| `pilot_services` | Anzahl Services im Mesh | Wachstum beobachten |
| `pilot_xds_config_size_bytes` | Größe der xDS-Config | Sidecar-Scope prüfen wenn hoch |

#### 4.2 Envoy-Metriken pro Proxy

```bash
## Config-Dump eines Sidecars ansehen
kubectl exec -it deploy/my-app -c istio-proxy -- \
  pilot-agent request GET config_dump | jq '.configs | length'

## Cluster-Anzahl (= bekannte Upstream-Services)
kubectl exec -it deploy/my-app -c istio-proxy -- \
  pilot-agent request GET clusters | wc -l
```

Wenn ein einzelner Sidecar hunderte Cluster hat, ist die `Sidecar`-Resource nicht richtig konfiguriert.

#### 4.3 Grafana Dashboards

Istio liefert vorkonfigurierte Dashboards:

- **Istio Control Plane Dashboard**: istiod CPU, Memory, Push-Zeiten, xDS-Connections
- **Istio Mesh Dashboard**: Überblick über alle Services und deren Traffic
- **Istio Performance Dashboard**: Proxy-Latenz und Ressourcenverbrauch

---

### 5. Best Practices Zusammenfassung

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

### 6. Hands-on Übung: Sidecar-Scope konfigurieren

#### Ausgangslage

Cluster mit drei Namespaces: `frontend`, `backend`, `database`. Jeder Sidecar sieht aktuell alle Services.

#### Aufgabe

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

#### Erwartetes Ergebnis

Die Cluster-Anzahl pro Sidecar sollte um 50–80% sinken, je nach Mesh-Größe.

### Performance-Optimierung


### Überblick

Istio fügt durch die Envoy-Proxies einen gewissen Overhead hinzu – Latenz, CPU, Memory und I/O. Ziel der Performance-Optimierung ist es, diesen Overhead so gering wie möglich zu halten, ohne auf die Vorteile des Service Mesh zu verzichten.

---

### 1. Proxy-Latenz verstehen und messen

Jeder Envoy-Sidecar fügt Latenz hinzu – **typisch ~0,5ms pro Hop**, bei Ausreißern (z.B. TLS-Handshake, xDS-Update, hohe Last) bis zu 1–3ms. Ein Hop = ein Proxy-Durchgang. Da der Traffic sowohl den Sender- als auch den Empfänger-Sidecar passiert, kommen pro Service-Call ca. 1–2ms dazu (bei Ausreißern bis zu 6ms).

Bei langen Call-Chains summiert sich das:

```
Client → Frontend → Backend → Database
         +1-2ms     +1-2ms    +1-2ms
                                      = 3–6ms zusätzliche Latenz (typ.)
```

**Latenz messen mit Istio-Metriken:**

```promql
## p99 Latenz pro Service (server-side)
## p99 = 99. Perzentil: 99% aller Requests sind schneller als dieser Wert
histogram_quantile(0.99, 
  sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m])) 
  by (le, destination_service))

## Differenz zwischen client- und server-reported Latenz = Proxy-Overhead
histogram_quantile(0.99, rate(istio_request_duration_milliseconds_bucket{reporter="source"}[5m]))
-
histogram_quantile(0.99, rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
```

---

### 2. Protocol Detection vs. explizites appProtocol

Standardmäßig muss Envoy das Protokoll jeder eingehenden Verbindung erraten (Protocol Sniffing). Das kostet Zeit – Envoy wartet kurz, um die ersten Bytes zu inspizieren.

**Lösung:** Protokoll explizit angeben, dann entfällt die Sniffing-Phase:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api
spec:
  ports:
    - name: http       
      port: 80
      targetPort: 8080
      appProtocol: http    # explizit – spart Sniffing-Phase
```

Alternativ über die Port-Naming-Konvention: `http-`, `grpc-`, `tcp-` als Prefix.

> **Empfehlung:** Immer `appProtocol` setzen. Das ist die zuverlässigste Methode und spart pro Verbindung einige Millisekunden Initialisierung.

---

### 3. Access Logging tunen

Per Default loggt Envoy jeden einzelnen Request. Bei hohem RPS (Requests Per Second) erzeugt das erhebliche CPU- und I/O-Last.

#### Wie Access Logging funktioniert

Access Logs sind die Logs der **Envoy-Proxies** – nicht die Anwendungslogs. Sie zeigen: wer hat wen aufgerufen, mit welchem Ergebnis. In `kubectl logs` sieht man beide, aber aus verschiedenen Containern:

```bash
## App-Logs
kubectl logs deploy/my-app -c my-app

## Envoy Access Logs
kubectl logs deploy/my-app -c istio-proxy
```

#### Konfiguration

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
## Schritt 1: Ziel und Format konfigurieren
meshConfig:
  accessLogFile: /dev/stdout    # wohin (stdout = sichtbar via kubectl logs)
  accessLogEncoding: JSON       # format
```

```yaml
## Schritt 2: Filter setzen – nur Fehler loggen
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

#### Mehrere Log-Ziele

Man kann zu verschiedenen Zielen mit unterschiedlichen Filtern loggen:

```yaml
## Externe Provider in meshConfig registrieren
meshConfig:
  accessLogFile: /dev/stdout
  extensionProviders:
    - name: otel-logging
      envoyOtelAls:
        service: otel-collector.monitoring.svc.cluster.local
        port: 4317
```

```yaml
## Verschiedene Filter pro Ziel
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

### 4. Tracing Sampling

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

### 5. Metrics Cardinality begrenzen

#### Was ist Metrics Cardinality?

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

#### Warum ist die Anzahl der Labels entscheidend?

Jedes zusätzliche Label **multipliziert** die Anzahl der möglichen Kombinationen:

```
10 Sources × 10 Destinations                              = 100 Zeitreihen
10 Sources × 10 Destinations × 5 Response-Codes           = 500 Zeitreihen
10 Sources × 10 Destinations × 5 Codes × 2 Protocols      = 1.000 Zeitreihen
10 Sources × 10 Destinations × 5 Codes × 2 Prot. × 2 Sec. = 2.000 Zeitreihen
```

Entfernt man `request_protocol` und `connection_security_policy`, fällt man von 2.000 auf 500 Zeitreihen – ein Viertel. Und das pro Metrik. Bei 5 Istio-Metriken spart man 6.000 Zeitreihen.

#### Hohe Cardinality = Problem für Prometheus

- Mehr Memory-Verbrauch (jede Zeitreihe lebt im RAM)
- Langsamere Queries
- Mehr Disk I/O

Bei 200 Services im Mesh können schnell Millionen von Zeitreihen entstehen.

#### Labels entfernen

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

### 6. Envoy Concurrency

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

### 7. mTLS-Overhead

Der TLS-Handshake und die Ver-/Entschlüsselung kosten CPU, aber in der Praxis ist der Overhead gering – typisch **< 0,5ms zusätzliche Latenz** und **~5–10% mehr CPU** auf dem Proxy.

Relevant wird es bei:

- Sehr hohem Throughput (>10.000 RPS pro Pod)
- Großen Payloads (Verschlüsselung skaliert mit Datenmenge)
- Ressourcen-knappe Umgebungen (Edge, IoT)

> **Empfehlung:** mTLS immer auf STRICT lassen. Der Sicherheitsgewinn überwiegt den minimalen Performance-Overhead bei weitem. Optimierung an anderen Stellen (Sidecar-Scope, Tracing, Logging) bringt deutlich mehr.

---

### 8. Connection Pooling: HTTP/2 vs. HTTP/1.1

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

### 9. Custom Extensions: EnvoyFilter, Lua und WasmPlugin

Wenn man das Verhalten der Envoy-Proxies anpassen will (z.B. Header hinzufügen, Requests transformieren), gibt es drei Ansätze.

#### 9.1 EnvoyFilter (native Config-Patches)

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

#### 9.2 Lua (via EnvoyFilter)

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

#### 9.3 WasmPlugin (empfohlener Weg)

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

#### Performance-Vergleich

| Ansatz | Latenz-Overhead | Memory | Upgrade-Stabilität |
|--------|----------------|--------|--------------------|
| **EnvoyFilter (native)** | Minimal | Gering | Fragil – bricht häufig |
| **Lua (via EnvoyFilter)** | Niedrig (~0,1–0,5ms) | Gering | Script stabil, EnvoyFilter-Wrapper fragil |
| **WasmPlugin** | Mittel (~0,5–2ms) | Höher (VM) | Stabil – eigene API |

#### Empfehlung

- **WasmPlugin** als Standard für Custom Extensions – stabil, sicher, portabel
- **Lua** für schnelle, einfache Eingriffe (Header-Manipulation, kleine Transformationen)
- **EnvoyFilter (native Config-Patches)** nur als letztes Mittel, wenn es keine andere Option gibt

---

### 10. Best Practices Zusammenfassung

1. **appProtocol** immer explizit setzen – spart Protocol-Sniffing-Overhead
2. **Access Logging** auf Fehler beschränken oder deaktivieren – spart CPU/IO
3. **Tracing Sampling** in Produktion auf 0,1–1% setzen – häufigste Ursache für unerklärlich hohen CPU-Verbrauch
4. **Metrics Cardinality** bewusst begrenzen – Labels entfernen die nicht abgefragt werden
5. **Envoy Concurrency** nur bei High-Throughput-Workloads erhöhen
6. **mTLS auf STRICT lassen** – Overhead minimal, Sicherheitsgewinn groß
7. **HTTP/2 zwischen Sidecars** aktivieren bei vielen parallelen Requests
8. **WasmPlugin** als Standard für Custom Extensions – Lua für kleine Eingriffe, EnvoyFilter (native) vermeiden

### Ambient Mesh Graphical Overview (sidecar-less Istio für Performance-Gewinn)


### Light: Only Layer 4 per node (ztunnel) 

  * No sidecar (envoy-proxy) per Pod, but one ztunnel agent per Node (Layer 4)
  * Enables security features (mtls, traffic encryption)

#### Like so:

  ![image](https://github.com/user-attachments/assets/755931d7-5bdd-43c9-8f93-28e8ee0b2bf3)

### Full fledged: Layer 4 (ztunnel) per Node & Layer 7 per Namespace (

  * One waypoint - proxy is rolled out per Namespace, which connects to the ztunnel agents 

![image](https://github.com/user-attachments/assets/a2aadab7-2ec0-446f-a35a-e972b8ac46b8)

#### Features in "fully-fledged" - ambient - mode 

![image](https://github.com/user-attachments/assets/30b89a37-cb71-46e9-a395-aafb593ebb12)


### Advantages:

  * Less overhead
  * One can start step-by-step moving towards a mesh (Layer 4 firstly and if wanted Layer 7 for specicfic namespaces)
  * Old pattern: sidecar and new pattern: ambient can live side by side. 

### Vergleich ohne istio, istio sidecar, istio ambient


  * https://livewyer.io/blog/2024/06/06/comparison-of-service-meshes-part-two/
  * https://github.com/livewyer-ops/poc-servicemesh2024/blob/main/docs/test-report-istio.md

## Erweiterte Routing-Techniken & Traffic-Optimierung (Istio API) 

### Canary Releases & Progressive Deployments / Traffic - Shifting


#### 0. Vorbereitung

```bash
mkdir -p ~/manifests/traffic-shifting
cd ~/manifests/traffic-shifting 

## Die Destinationen-Versionen anlegen
cp -a ~/istio/samples/bookinfo/networking/destination-rule-all.yaml destination-rule-all.yaml
kubectl -n bookinfo apply -f destination-rule-all.yaml 
```
#### 1. 100% Traffic -> reviews.v1 

```
cat ~/istio/samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/networking/virtual-service-all-v1.yaml
```

```
kubectl get vs -n bookinfo reviews -o yaml | head -n 30
```

#### 2. Testen 

```
## Seite öffnen
http://<deine-ip>/productpage

## Egal wie oft du die Seite lädst, es bleibt immer v1
```

#### 3. 50% (v1) /50% (v3) Traffic

```
cat ~/istio/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

```bash
kubectl -n bookinfo get vs reviews -o yaml 
```

### 4. Testen 


```
## Seite öffnen
http://<deine-ip>/productpage

## Abwechselnd bei mehrmals laden v1 (keine Sterne) und v3 (sterne)
```

### 5. 100% auf v3 

```
cat ~/istio/samples/bookinfo/networking/virtual-service-reviews-v3.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```


```bash
$BOOKINFO_URL="deine-ip"
for i in {1..10}; do
  curl -fsS "$BOOKINFO_URL/productpage" \
    | grep -c 'glyphicon-star"' \
    | awk '{print $1" Sterne"}'
done
```

#### 6. Aufräumen 

```
kubectl delete -n bookinfo vs reviews
```

### Reference:

 * https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/

## Installation von istio (ambient mode)

### istio-installation mit istioctl - ambient


>[NOTE:]
>you need to adjust error from calico with 
>kubectl edit felixconfigurations default -o yaml

### Step 1: Installation and CRDs

```
istioctl install --set profile=ambient --skip-confirmation
## In addition you will need the gateway api - crd's
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard.yaml
```

### istio-installation with helm - ambient


#### Voraussetzung

- Helm-Charts `base`, `cni` und `istiod` bereits installiert (Version 1.29.1)
- Helm-Profile ambient: https://github.com/istio/istio/blob/master/manifests/helm-profiles/ambient.yaml

#### Step 1: Gateway API CRDs (v1.4.0 - Standard)
```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

#### Step 2: istiod und cni auf Ambient-Profil upgraden
```bash
## Zur Sicherheit, manchmal gibt es Probleme mit dem Mutating Webhook
kubectl delete validatingwebhookconfigurations istio-validator-istio-system

helm upgrade istiod istio/istiod -n istio-system --set profile=ambient --version 1.29.1 
helm upgrade istio-cni istio/cni -n istio-system --set profile=ambient --version 1.29.1 
```

#### Step 3: ztunnel installieren
```bash
helm install ztunnel istio/ztunnel \
  --namespace istio-system \
  --version 1.29.1 \
  --wait
```

### Bookinfo demo in ambient ausrollen


### Überblick

<img width="693" height="465" alt="image" src="https://github.com/user-attachments/assets/22cbf386-5a90-458b-8157-51620ef829ea" />

### (Optional) Alte Installation entfernen 

```
kubectl delete ns bookinfo
```

### Vorbereitung

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

### Waypoint Proxy ausrollen

```
cd
mkdir -p manifests/waypoint
cd manifests/waypoint
```  


```
## YAML generieren (dry-run)
istioctl waypoint generate --namespace bookinfo --for all  > waypoint.yaml 
```

```
## Anschauen was passiert
cat waypoint.yaml 
```

```
## Ausrollen
kubectl apply -f waypoint.yaml
kubectl label namespace bookinfo istio.io/use-waypoint=waypoint
```

```
## Überprüfen
kubectl -n bookinfo get gateways
istioctl waypoint list --namespace bookinfo
```

### Bookinfo App ausrollen

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml
kubectl -n bookinfo get all
```

### Testen ob die App funktioniert

```
kubectl -n bookinfo exec "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

```
## Einfachere Variante
kubectl -n bookinfo exec deployments/ratings-v1 -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

### App mit Gateway API nach aussen öffnen

```
## That's what we do ....
cat ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl -n bookinfo get gateways
kubectl -n bookinfo get httproutes -o yaml
```

```
## Die external-ip aus diesem Output notieren
## Das Gateway erstellt automatisch einen Service
kubectl -n bookinfo get svc bookinfo-gateway-istio
```

```
http://<external-ip>/productpage
## oder im Browser öffnen
```

### Ref

  * https://istio.io/latest/docs/ambient/getting-started/deploy-sample-app/

## Erweiterbarkeit von Istio

### Wo läuft WASM (WebAssembly) im Rahmen von istio ?


  * Direkt in envoy eingebuat 

**WASM-Runtime in Envoy:**
- Envoy hat eine eingebettete WASM-Runtime (meist **V8** oder **Wasmtime**)
- Läuft im gleichen Prozess wie Envoy selbst
- WASM-Module werden zur Laufzeit geladen und im Sandbox ausgeführt

**Architektur:**
```
Pod
├── App-Container
└── Istio-Sidecar (Envoy)
    └── WASM-Runtime (V8/Wasmtime)
        └── Dein WASM-Plugin
```

**Vorteile dieser Integration:**
- Kein extra Container nötig
- Schnelle Kommunikation (keine Netzwerk-Calls)
- Memory-isoliert durch WASM-Sandbox
- Hot-Reload möglich ohne Envoy-Neustart

**Runtime-Optionen:**
- **V8** (Standard): Von Chrome, gut getestet
- **Wasmtime**: Leichtgewichtiger, Rust-basiert
- **WAVM**: Veraltet, nicht mehr empfohlen

Die Runtime wird beim Envoy-Build festgelegt - als User kannst du das normalerweise nicht ändern.

### Eigene Istio-Erweiterungen mit WebAssembly schreiben (RateLimit-Plugin)


### Lernziele

- WASM-Plugin in Go für Envoy/Istio entwickeln
- Token-Bucket-Algorithmus implementieren
- WasmPlugin-Resource konfigurieren (Sidecar, Ambient, Gateway)

### Voraussetzungen

- Kubernetes-Cluster mit Istio ≥ 1.22
- Container-Registry (Harbor, GHCR, Docker Hub)

---

### 1 — Tooling installieren

```bash
## Go (https://go.dev/dl/)
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

## TinyGo — WASM-fähiger Go-Compiler (Standard-Go kann kein wasi-Target)
wget https://github.com/tinygo-org/tinygo/releases/download/v0.33.0/tinygo_0.33.0_amd64.deb
sudo dpkg -i tinygo_0.33.0_amd64.deb

## ORAS — OCI-Registry-Tool zum Pushen von WASM-Modulen
curl -LO https://github.com/oras-project/oras/releases/download/v1.2.0/oras_1.2.0_linux_amd64.tar.gz
tar xzf oras_1.2.0_linux_amd64.tar.gz && sudo mv oras /usr/local/bin/

## Prüfen
go version && tinygo version && oras version
```

---

### 2 — Projekt anlegen

```bash
mkdir -p istio-ratelimit-wasm && cd istio-ratelimit-wasm
go mod init github.com/t3company/istio-ratelimit-wasm
go get github.com/tetratelabs/proxy-wasm-go-sdk
```

---

### 3 — Token Bucket Algorithmus

```
 Tokens: ████████░░  (8/10)

   Request → Token verbrauchen → 7/10 → 200 OK
   Request → Token verbrauchen → 6/10 → 200 OK
   ...
   Request → Bucket leer       → 0/10 → 429!

   Timer Tick (1s) → Refill +10 → 10/10
```

| Parameter | Beschreibung |
|-----------|-------------|
| `max_tokens` | Burst-Kapazität |
| `refill_rate` | Tokens/Sekunde nachgefüllt |

Vorteil gegenüber Fixed Window: kein Boundary-Problem, Burst-fähig, glättet langfristigen Durchsatz.

---

### 4 — Plugin implementieren

#### Proxy-WASM Architektur (Go)

```
VMContext → PluginContext → HttpContext
                │                │
                ├─ OnPluginStart()          Config + Timer
                ├─ OnTick()                 Token Refill
                └─ NewHttpContext()
                         └─ OnHttpRequestHeaders()  Token prüfen
```

#### main.go

```go
package main

import (
	"encoding/binary"
	"encoding/json"
	"fmt"

	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm"
	"github.com/tetratelabs/proxy-wasm-go-sdk/proxywasm/types"
)

func main() { proxywasm.SetVMContext(&vmContext{}) }

// ── Config ──────────────────────────────────────────────────────────────────

type rateLimitConfig struct {
	MaxTokens        uint64 `json:"max_tokens"`
	RefillRate       uint64 `json:"refill_rate"`
	RefillIntervalMs uint64 `json:"refill_interval_ms"`
	StatusCode       uint32 `json:"status_code"`
	ResponseBody     string `json:"response_body"`
}

func defaultConfig() rateLimitConfig {
	return rateLimitConfig{100, 10, 1000, 429, "rate limit exceeded"}
}

// ── Shared Data Keys + Helpers ──────────────────────────────────────────────

const (
	bucketKey       = "ratelimit_tokens"
	statsAllowedKey = "ratelimit_allowed"
	statsDeniedKey  = "ratelimit_denied"
)

func toBytes(v uint64) []byte {
	b := make([]byte, 8)
	binary.LittleEndian.PutUint64(b, v)
	return b
}

func fromBytes(b []byte) uint64 {
	if len(b) < 8 { return 0 }
	return binary.LittleEndian.Uint64(b)
}

func incrementStat(key string) {
	data, cas, err := proxywasm.GetSharedData(key)
	if err != nil { return }
	proxywasm.SetSharedData(key, toBytes(fromBytes(data)+1), cas)
}

// ── VM Context ──────────────────────────────────────────────────────────────

type vmContext struct{ types.DefaultVMContext }

func (*vmContext) NewPluginContext(contextID uint32) types.PluginContext {
	return &pluginContext{config: defaultConfig()}
}

// ── Plugin Context (Config + Timer + Bucket) ────────────────────────────────

type pluginContext struct {
	types.DefaultPluginContext
	config rateLimitConfig
}

func (p *pluginContext) OnPluginStart(configSize int) types.OnPluginStartStatus {
	if configSize > 0 {
		if data, err := proxywasm.GetPluginConfiguration(); err == nil {
			var cfg rateLimitConfig
			if err := json.Unmarshal(data, &cfg); err == nil {
				if cfg.MaxTokens == 0        { cfg.MaxTokens = 100 }
				if cfg.RefillRate == 0       { cfg.RefillRate = 10 }
				if cfg.RefillIntervalMs == 0 { cfg.RefillIntervalMs = 1000 }
				if cfg.StatusCode == 0       { cfg.StatusCode = 429 }
				if cfg.ResponseBody == ""    { cfg.ResponseBody = "rate limit exceeded" }
				p.config = cfg
			}
		}
	}

	proxywasm.LogInfof("Rate Limiter: max_tokens=%d, refill_rate=%d/s",
		p.config.MaxTokens, p.config.RefillRate)

	proxywasm.SetSharedData(bucketKey, toBytes(p.config.MaxTokens), 0)
	proxywasm.SetSharedData(statsAllowedKey, toBytes(0), 0)
	proxywasm.SetSharedData(statsDeniedKey, toBytes(0), 0)
	proxywasm.SetTickPeriodMilliSeconds(uint32(p.config.RefillIntervalMs))
	return types.OnPluginStartStatusOK
}

func (p *pluginContext) OnTick() {
	data, cas, err := proxywasm.GetSharedData(bucketKey)
	if err != nil { return }

	current := fromBytes(data)
	add := p.config.RefillRate * p.config.RefillIntervalMs / 1000
	newTokens := current + add
	if newTokens > p.config.MaxTokens { newTokens = p.config.MaxTokens }

	if err := proxywasm.SetSharedData(bucketKey, toBytes(newTokens), cas); err != nil {
		proxywasm.LogWarn("refill CAS conflict, retry next tick")
	}
}

func (p *pluginContext) NewHttpContext(contextID uint32) types.HttpContext {
	return &httpContext{config: p.config}
}

// ── HTTP Context (pro Request) ──────────────────────────────────────────────

type httpContext struct {
	types.DefaultHttpContext
	config rateLimitConfig
}

func (h *httpContext) OnHttpRequestHeaders(numHeaders int, endOfStream bool) types.Action {
	for attempt := 0; attempt < 3; attempt++ {
		data, cas, err := proxywasm.GetSharedData(bucketKey)
		if err != nil { continue }

		current := fromBytes(data)

		if current == 0 {
			incrementStat(statsDeniedKey)
			proxywasm.SendHttpResponse(h.config.StatusCode, [][2]string{
				{"x-ratelimit-limit", fmt.Sprintf("%d", h.config.MaxTokens)},
				{"x-ratelimit-remaining", "0"},
				{"retry-after", "1"},
			}, []byte(h.config.ResponseBody), -1)
			return types.ActionPause
		}

		if err := proxywasm.SetSharedData(bucketKey, toBytes(current-1), cas); err != nil {
			continue // CAS-Konflikt → retry
		}

		incrementStat(statsAllowedKey)
		proxywasm.AddHttpRequestHeader("x-ratelimit-remaining", fmt.Sprintf("%d", current-1))
		return types.ActionContinue
	}

	proxywasm.LogWarn("CAS exhausted, fail-open")
	return types.ActionContinue
}
```

#### Wichtige Konzepte

- **CAS (Compare-And-Swap):** `GetSharedData` liefert Wert + CAS-Token. `SetSharedData` schreibt nur wenn Token noch gültig. Bei Konflikt → retry.
- **Fail-Open:** Nach 3 CAS-Retries wird der Request durchgelassen — kaputtes Plugin soll nicht allen Traffic blockieren.
- **`types.DefaultHttpContext`:** Go-Embedding, implementiert alle Interface-Methoden als No-Ops. Wir überschreiben nur was wir brauchen.

---

### 5 — Build & Push

```bash
## Kompilieren (-scheduler=none: kein Goroutine-Scheduler, -target=wasi: WASM)
tinygo build -o ratelimit.wasm -scheduler=none -target=wasi main.go

ls -lh ratelimit.wasm   # ~500 KB - 1.5 MB

## Optional: ~30% kleiner
wasm-opt -Os ratelimit.wasm -o ratelimit.wasm

## Pushen
oras login registry.example.com
oras push registry.example.com/wasm/ratelimit:v1 \
  ratelimit.wasm:application/vnd.module.wasm.content.layer.v1+wasm
```

---

### 6 — Deployen

```bash
kubectl create namespace demo
kubectl label namespace demo istio-injection=enabled
kubectl apply -f k8s/test-deployment.yaml
kubectl -n demo get pods -w   # Warten auf 2/2 Running
```

`k8s/test-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: demo
spec:
  selector:
    app: my-service
  ports:
    - port: 8080
      targetPort: 80
```

`k8s/wasmplugin.yaml`:

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: local-ratelimit
  namespace: demo
spec:
  selector:
    matchLabels:
      app: my-service
  url: oci://registry.example.com/wasm/ratelimit:v1
  phase: AUTHN                    # Vor Authentication in der Filterkette
  pluginConfig:
    max_tokens: 100
    refill_rate: 10
    refill_interval_ms: 1000
    status_code: 429
    response_body: '{"error": "rate limit exceeded", "retry_after": 1}'
  match:
    - mode: SERVER                # Nur Inbound
      ports:
        - number: 8080
  imagePullPolicy: IfNotPresent
```

```bash
kubectl apply -f k8s/wasmplugin.yaml
kubectl -n demo logs deploy/my-service -c istio-proxy | grep -i "rate"
```

---

### 7 — Testen

```bash
## Einzelner Request
kubectl -n demo run test --rm -it --image=curlimages/curl -- \
  curl -sv http://my-service:8080/get 2>&1 | grep -E "HTTP|x-ratelimit"

## Load Test
kubectl -n demo run loadtest --rm -it --image=curlimages/curl -- sh -c '
  ALLOWED=0; DENIED=0
  for i in $(seq 1 200); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}\n" http://my-service:8080/get)
    [ "$STATUS" = "429" ] && DENIED=$((DENIED + 1)) || ALLOWED=$((ALLOWED + 1))
  done
  echo "Erlaubt: $ALLOWED | Gedrosselt: $DENIED | Gesamt: 200"
'
```

**Erwartet:** ~100 erlaubt (Burst), Rest 429. Pro Sekunde kommen ~10 nach.

---

### 8 — Config ändern

`max_tokens: 10`, `refill_rate: 2` setzen → `kubectl apply` → Load Test wiederholen. Erwartet: nur ~10 erlaubt.

> **Frage:** Was passiert bei 3 Replicas? → Jeder Pod hat eigenen Bucket → 3× Durchsatz. Das ist Local vs. Global Rate Limiting.

---

### 9 — Ambient Mode

```bash
kubectl label namespace demo istio-injection-
kubectl label namespace demo istio.io/dataplane-mode=ambient
istioctl waypoint apply -n demo --name my-service-waypoint --for service
kubectl -n demo label service my-service istio.io/use-waypoint=my-service-waypoint
```

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: local-ratelimit-ambient
  namespace: demo
spec:
  targetRefs:                       # statt selector
    - kind: Service
      group: ""
      name: my-service
  url: oci://registry.example.com/wasm/ratelimit:v1
  phase: AUTHN
  pluginConfig:
    max_tokens: 50
    refill_rate: 5
    refill_interval_ms: 1000
```

Waypoint Proxy = ein zentraler Proxy pro Service → ein Bucket → verhält sich wie Global Rate Limiting.

---

### 10 — Gateway-Level Rate Limiting

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
  name: gateway-ratelimit
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  url: oci://registry.example.com/wasm/ratelimit:v1
  phase: AUTHN
  pluginConfig:
    max_tokens: 1000
    refill_rate: 100
```

Gateway schützt den Cluster, Service-Level schützt einzelne Microservices. In Produktion beide kombinieren.

---

### 11 — Bonus-Aufgaben

**Per Source-IP:** `proxywasm.GetHttpRequestHeader("x-forwarded-for")` → Bucket-Key pro IP.

**Health-Checks ausnehmen:** `:path` Header prüfen, bei `/health` direkt `ActionContinue`.

**Per Pfad:** Separate Config pro Pfad-Prefix in `pluginConfig`.

---

### Vergleich

| | EnvoyFilter | WasmPlugin |
|--|-------------|------------|
| API-Stabilität | "break glass" | Stabile Istio-API |
| Portierbarkeit | Istio-only | Envoy, Istio, Gateway API |
| Ambient Mode | Nein | Ja |

| | Go (TinyGo) | Rust |
|--|-------------|------|
| Lernkurve | Flach | Steil |
| Binary-Größe | ~500 KB - 1.5 MB | ~200-400 KB |
| Einschränkungen | Kein `net/http`, kein `reflect` | Keine |
| Empfehlung | Prototyping, Go-Teams | Produktion, Performance |

---

### Aufräumen

```bash
kubectl -n demo delete wasmplugin --all && kubectl delete namespace demo
```

### Referenzen

- [Proxy-WASM Go SDK](https://github.com/tetratelabs/proxy-wasm-go-sdk) · [TinyGo](https://tinygo.org/) · [Istio WasmPlugin API](https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/) · [ORAS](https://oras.land/docs/)

## Automatisierung mit GitOps & ArgoCD

### Was ist ArgoCD


### 1. Architektur-Überblick

#### Istio Ambient Mode – Komponenten

Im Gegensatz zum Sidecar-Modus gibt es bei Ambient Mode **kein Envoy-Sidecar** pro Pod.
Stattdessen wird die Mesh-Funktionalität in zwei Schichten aufgeteilt:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Control Plane (istio-system)               │  │
│  │                                                               │  │
│  │   ┌─────────┐     ┌──────────────┐     ┌──────────────────┐  │  │
│  │   │ istiod  │     │  istio-cni   │     │   Gateway API    │  │  │
│  │   │ (xDS)   │     │  (DaemonSet) │     │     CRDs         │  │  │
│  │   └────┬────┘     └──────────────┘     └──────────────────┘  │  │
│  │        │  xDS-Config                                          │  │
│  └────────┼──────────────────────────────────────────────────────┘  │
│           │                                                         │
│  ┌────────┼──────────────────────────────────────────────────────┐  │
│  │  L4    │   Data Plane – ztunnel (DaemonSet, pro Node)        │  │
│  │        ▼                                                      │  │
│  │   ┌─────────┐          ┌─────────┐          ┌─────────┐     │  │
│  │   │ ztunnel │          │ ztunnel │          │ ztunnel │     │  │
│  │   │ Node 1  │◄─HBONE──►│ Node 2  │◄─HBONE──►│ Node 3  │     │  │
│  │   └────┬────┘          └────┬────┘          └────┬────┘     │  │
│  │        │                    │                    │           │  │
│  └────────┼────────────────────┼────────────────────┼───────────┘  │
│           │                    │                    │               │
│  ┌────────┼────────────────────┼────────────────────┼───────────┐  │
│  │  L7    │   Optional: Waypoint Proxies (Envoy, pro Namespace) │  │
│  │        ▼                    ▼                                 │  │
│  │   ┌──────────┐        ┌──────────┐                           │  │
│  │   │ Waypoint │        │ Waypoint │   ← Nur bei L7-Bedarf    │  │
│  │   │ ns: app  │        │ ns: web  │     (AuthzPolicy, Retry) │  │
│  │   └──────────┘        └──────────┘                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Application Pods (KEIN Sidecar!)                            │   │
│  │   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐             │   │
│  │   │App A│  │App B│  │App C│  │App D│  │App E│             │   │
│  │   └─────┘  └─────┘  └─────┘  └─────┘  └─────┘             │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Zusammenfassung der Komponenten:**

| Helm Chart   | Funktion                                  | Typ        |
|-------------|-------------------------------------------|------------|
| `istio/base`  | CRDs (VirtualService, DestinationRule...) | Cluster    |
| `istio/istiod`| Control Plane (xDS, Cert-Mgmt)           | Cluster    |
| `istio/cni`   | CNI Plugin, Traffic-Redirect zu ztunnel  | DaemonSet  |
| `istio/ztunnel`| L4-Proxy, mTLS, HBONE-Tunnel            | DaemonSet  |
| Gateway API   | CRDs für Waypoint/Ingress-Konfiguration  | Cluster    |


---

### 2. Wie ArgoCD mit Istio Ambient zusammenspielt

#### GitOps-Workflow

```
┌──────────────┐    Push     ┌──────────────┐    Sync     ┌──────────────┐
│              │────────────►│              │────────────►│              │
│   Developer  │             │   Git Repo   │             │   ArgoCD     │
│              │◄────────────│              │◄────────────│   Controller │
│              │    PR/Review│  (Source of   │   Diff/     │              │
└──────────────┘             │   Truth)     │   Status    └──────┬───────┘
                             └──────────────┘                    │
                                                                 │ Helm
                                                                 │ Template
                                                                 │ + Apply
                                                                 ▼
                             ┌────────────────────────────────────────────┐
                             │          Kubernetes Cluster                │
                             │                                            │
                             │  ┌─────────┐ ┌────────┐ ┌───────────┐    │
                             │  │istio-   │ │istiod  │ │ztunnel    │    │
                             │  │base     │ │        │ │+ istio-cni│    │
                             │  │(CRDs)   │ │(CP)    │ │(DP)       │    │
                             │  └─────────┘ └────────┘ └───────────┘    │
                             │                                            │
                             │  ┌──────────────────────────────────┐     │
                             │  │  App-Namespace (ambient labeled) │     │
                             │  │  ┌──────┐ ┌──────┐ ┌──────────┐ │     │
                             │  │  │App A │ │App B │ │Waypoint  │ │     │
                             │  │  └──────┘ └──────┘ └──────────┘ │     │
                             │  └──────────────────────────────────┘     │
                             └────────────────────────────────────────────┘
```

#### Install-Reihenfolge (kritisch!)

ArgoCD deployed Helm Charts. Bei Istio Ambient gibt es **Abhängigkeiten**:

```
  Sync Wave 0        Sync Wave 1       Sync Wave 2        Sync Wave 3
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Gateway API │  │  istio-base  │  │   istiod     │  │  istio-cni   │
│  CRDs        │──►│  (CRDs)      │──►│  (profile:   │──►│  (profile:   │
│              │  │              │  │   ambient)   │  │   ambient)   │
└──────────────┘  └──────────────┘  └──────┬───────┘  └──────┬───────┘
                                           │                  │
                                           │                  ▼
                                           │          ┌──────────────┐
                                           └─────────►│   ztunnel    │
                                                      │              │
                                                      └──────────────┘
                                                              │
                                                              ▼
                                                      ┌──────────────┐
                                                      │  Apps +      │
                                                      │  Namespace   │
                                                      │  Labeling    │
                                                      └──────────────┘
```

**Reihenfolge:** Gateway API CRDs → istio-base → istiod → cni → ztunnel → Apps

---

### 3. Gotchas & Fallstricke

#### NetworkPolicy + HBONE Port 15008

Das ist der häufigste Fehler! Ambient Mode nutzt HBONE (HTTP/2 + mTLS) über **Port 15008**.
Bestehende NetworkPolicies blockieren diesen Port oft.

```
  PROBLEM:                                  LÖSUNG:
┌───────────────┐                        ┌───────────────┐
│  NetworkPolicy│                        │  NetworkPolicy│
│               │                        │               │
│  ingress:     │                        │  ingress:     │
│  - port: 8080 │  ← blockiert 15008!   │  - port: 8080 │
│               │                        │  - port: 15008│  ← HBONE erlauben!
└───────────────┘                        └───────────────┘
```

#### ArgoCD HA + Ambient Mode

ArgoCD HA nutzt Redis-HA mit Sentinel. Wenn der ArgoCD-Namespace ins Ambient Mesh
eingebunden wird, muss Port 15008 in **allen** ArgoCD-NetworkPolicies erlaubt werden.

**Empfehlung:** ArgoCD-Namespace **NICHT** ins Ambient Mesh einbinden – ArgoCD ist die Management-Ebene für Istio und muss unabhängig davon funktionieren.
ArgoCD verwaltet Istio – es muss nicht selbst im Mesh sein.


### 4. Initiales Ausrollen – Schritt für Schritt

```
Was Du tust (manuell)              Was ArgoCD tut (automatisch)
─────────────────────              ────────────────────────────

Schritt 1: Cluster + ArgoCD
  bereitstellen
         │
         ▼
Schritt 2: Git-Repo anlegen
  mit allen Application-YAMLs
         │
         ▼
Schritt 3: root-app.yaml
  kubectl apply (1× manuell!)
         │                         ArgoCD liest apps/ aus Git
         └────────────────────────►      │
                                         ├─► erstellt gateway-api-crds App
                                         ├─► erstellt istio-base App
                                         ├─► erstellt istiod App
                                         ├─► erstellt istio-cni App
                                         ├─► erstellt ztunnel App
                                         └─► erstellt bookinfo App
                                                     │
                                              Sync Waves steuern
                                              die Reihenfolge
                                                     │
                                                     ▼
                                              Istio Ambient läuft!

Ab jetzt: Änderungen NUR noch über Git-Commits.
```

#### Schritt 1: Voraussetzungen prüfen

```bash
## Kubernetes >= 1.28
kubectl version

## Helm >= 3.6 (nur zum Prüfen, ArgoCD nutzt Helm intern)
helm version
```

#### Schritt 2: ArgoCD installieren

```bash
## Falls noch kein ArgoCD im Cluster:
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## Warten bis alles läuft:
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

## Admin-Passwort holen:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

## Optional: Port-Forward für UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
## → https://localhost:8080  (User: admin)
```

#### Schritt 3: Git-Repo anlegen und befüllen

```bash
## Neues Repo erstellen (z.B. auf GitHub/GitLab)
mkdir istio-ambient-gitops && cd istio-ambient-gitops
git init

## Verzeichnisstruktur anlegen
mkdir -p apps/istio/values apps/bookinfo
```

Jetzt alle Application-YAMLs und Values-Dateien erstellen
(siehe Abschnitt 4.5 für den Inhalt jeder Datei):

```bash
## Dateien erstellen:
touch apps/istio/gateway-api-crds.yaml
touch apps/istio/istio-base-app.yaml
touch apps/istio/istiod-app.yaml
touch apps/istio/istio-cni-app.yaml
touch apps/istio/ztunnel-app.yaml
touch apps/istio/values/istiod-values.yaml
touch apps/bookinfo/bookinfo-app.yaml

## Alles committen und pushen:
git add -A
git commit -m "Initial Istio Ambient + ArgoCD setup"
git remote add origin https://github.com/DEIN-USER/istio-ambient-gitops.git
git push -u origin main
```

#### Schritt 4: root-app.yaml erstellen und anwenden

Diese Datei liegt NICHT im Git-Repo – sie ist der einzige manuelle Schritt:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

#### Schritt 5: Zuschauen und verifizieren

```bash
## ArgoCD Applications beobachten:
kubectl get applications -n argocd -w

## Erwartete Ausgabe nach ein paar Minuten:
## NAME                STATUS   HEALTH
## root                Synced   Healthy
## gateway-api-crds    Synced   Healthy
## istio-base          Synced   Healthy
## istiod              Synced   Healthy
## istio-cni           Synced   Healthy
## ztunnel             Synced   Healthy

## Istio-Pods prüfen:
kubectl get pods -n istio-system

## CRDs prüfen:
kubectl get crds | grep -E 'istio|gateway'
```

#### Ab jetzt: Alles über Git

```bash
## Beispiel: Neue App hinzufügen
vim apps/bookinfo/bookinfo-app.yaml
git add -A && git commit -m "Add bookinfo app"
git push
## → ArgoCD erkennt die Änderung und deployed automatisch

## Beispiel: Istio upgraden
## In apps/istio/*-app.yaml: targetRevision von 1.24.2 auf 1.25.0 ändern
git add -A && git commit -m "Upgrade Istio to 1.25.0"
git push
## → ArgoCD rolled die neuen Versionen aus
```

---

#### 4.4 Git-Repo Struktur

```
istio-ambient-gitops/
├── apps/
│   ├── istio/
│   │   ├── gateway-api-crds.yaml       # ArgoCD Application
│   │   ├── istio-base-app.yaml         # ArgoCD Application
│   │   ├── istiod-app.yaml             # ArgoCD Application
│   │   ├── istio-cni-app.yaml          # ArgoCD Application
│   │   ├── ztunnel-app.yaml            # ArgoCD Application
│   │   └── values/
│   │       ├── istiod-values.yaml
│   │       ├── cni-values.yaml
│   │       └── ztunnel-values.yaml
│   ├── bookinfo/
│   │   └── bookinfo-app.yaml           # ArgoCD Application
│   └── root-app.yaml                   # App-of-Apps
```

#### 4.5 ArgoCD Application Manifeste

##### Gateway API CRDs (Sync Wave -3)

```yaml
## apps/istio/gateway-api-crds.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-api-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/kubernetes-sigs/gateway-api.git
    targetRevision: v1.4.0
    path: config/crd/standard
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=false
```

##### istio-base (Sync Wave -2)

```yaml
## apps/istio/istio-base-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-base
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: base
    targetRevision: 1.24.2     # oder 1.29.x je nach Version
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true    # wichtig für CRDs!
```

##### istiod (Sync Wave -1)

```yaml
## apps/istio/istiod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istiod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: istiod
      targetRevision: 1.24.2
      helm:
        valueFiles:
          - $values/apps/istio/values/istiod-values.yaml
    - repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
```

**istiod-values.yaml:**
```yaml
## apps/istio/values/istiod-values.yaml
profile: ambient
pilot:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
meshConfig:
  accessLogFile: /dev/stdout
```

##### istio-cni (Sync Wave 0)

```yaml
## apps/istio/istio-cni-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-cni
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: cni
    targetRevision: 1.24.2
    helm:
      values: |
        profile: ambient
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system       # oder kube-system je nach Setup
  syncPolicy:
    automated:
      selfHeal: true
```

##### ztunnel (Sync Wave 1)

```yaml
## apps/istio/ztunnel-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ztunnel
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: ztunnel
    targetRevision: 1.24.2
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
```

##### Multi-Source Variante (alles in einer Application)

Alternativ kann man alle Charts in einer einzigen Application mit `sources` bündeln.
Das funktioniert, aber man verliert die Sync-Wave-Kontrolle:

```yaml
## Kompakte Variante – ACHTUNG: keine Reihenfolge-Garantie!
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-ambient
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  sources:
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: base
      targetRevision: 1.24.2
      helm:
        releaseName: istio-base
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: istiod
      targetRevision: 1.24.2
      helm:
        releaseName: istiod
        parameters:
          - name: profile
            value: ambient
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: cni
      targetRevision: 1.24.2
      helm:
        releaseName: istio-cni
        parameters:
          - name: profile
            value: ambient
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: ztunnel
      targetRevision: 1.24.2
      helm:
        releaseName: ztunnel
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

#### 4.6 App-of-Apps (Root Application)

```yaml
## apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
```

#### 4.7 Namespace ins Ambient Mesh einbinden

```yaml
## bookinfo-namespace.yaml (im Git-Repo!)
apiVersion: v1
kind: Namespace
metadata:
  name: bookinfo
  labels:
    istio.io/dataplane-mode: ambient    # ← Das ist der Schlüssel!
```

#### 4.8 Waypoint Proxy deployen (für L7-Features)

```yaml
## Nur nötig wenn AuthorizationPolicy, Retries, etc. auf L7 gebraucht werden
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-waypoint
  namespace: bookinfo
  labels:
    istio.io/waypoint-for: service      # oder "all" für den ganzen Namespace
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
```

#### 4.9 Verifizierung

```bash
## Istio-Komponenten prüfen
kubectl get pods -n istio-system
## Erwartung: istiod-xxx, ztunnel-xxx (pro Node), istio-cni-node-xxx (pro Node)

## CRDs prüfen
kubectl get crds | grep -E 'istio|gateway'

## Ambient-Enrollment prüfen
kubectl get namespace bookinfo --show-labels | grep dataplane-mode

## ztunnel-Logs: mTLS-Verbindungen sichtbar
kubectl logs -n istio-system -l app=ztunnel --tail=20

## Mesh-Status mit istioctl
istioctl ztunnel-config workloads
```

---

### 5. Checkliste: Was muss ich beachten?

```
 ✓  Gateway API CRDs VOR Istio installieren (Sync Wave!)
 ✓  Helm Charts in korrekter Reihenfolge: base → istiod → cni → ztunnel
 ✓  ServerSideApply=true für CRD-haltige Charts setzen
 ✓  profile: ambient bei istiod und cni setzen
 ✓  NetworkPolicies: Port 15008 (HBONE) freigeben
 ✓  ArgoCD-Namespace NICHT ins Ambient Mesh (Management-Ebene bleibt unabhängig)
 ✓  Namespace-Label: istio.io/dataplane-mode=ambient
 ✓  Waypoint nur deployen, wenn L7-Features nötig sind
 ✓  Alle Istio-Versionen einheitlich halten (base, istiod, cni, ztunnel)
 ✓  Bei Upgrades: targetRevision in Git ändern → ArgoCD synced automatisch
```

---

### 6. Upgrade-Workflow

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│ 1. Git Branch  │     │ 2. PR Review   │     │ 3. Merge       │
│                │     │                │     │                │
│ targetRevision │────►│ Diff prüfen:   │────►│ ArgoCD synced  │
│ 1.24.2 → 1.25.0│    │ - Release Notes│     │ automatisch    │
│                │     │ - Breaking Ch. │     │                │
└────────────────┘     └────────────────┘     └────┬───────────┘
                                                    │
                                        ┌───────────┼───────────┐
                                        │           │           │
                                        ▼           ▼           ▼
                                    base CRDs    istiod     ztunnel
                                    (update)    (rollout)  (rollout)
```

**Wichtig:** ztunnel ist ein DaemonSet – beim Upgrade gibt es einen kurzen
Traffic-Umbruch pro Node. In Produktion: `maxUnavailable: 1` in den Values setzen.

### istio ambient und bookinfo mit argocd ausrollen


### 1. Architektur-Überblick

#### Istio Ambient Mode – Komponenten

Im Gegensatz zum Sidecar-Modus gibt es bei Ambient Mode **kein Envoy-Sidecar** pro Pod.
Stattdessen wird die Mesh-Funktionalität in zwei Schichten aufgeteilt:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Control Plane (istio-system)               │  │
│  │                                                               │  │
│  │   ┌─────────┐     ┌──────────────┐     ┌──────────────────┐  │  │
│  │   │ istiod  │     │  istio-cni   │     │   Gateway API    │  │  │
│  │   │ (xDS)   │     │  (DaemonSet) │     │     CRDs         │  │  │
│  │   └────┬────┘     └──────────────┘     └──────────────────┘  │  │
│  │        │  xDS-Config                                          │  │
│  └────────┼──────────────────────────────────────────────────────┘  │
│           │                                                         │
│  ┌────────┼──────────────────────────────────────────────────────┐  │
│  │  L4    │   Data Plane – ztunnel (DaemonSet, pro Node)        │  │
│  │        ▼                                                      │  │
│  │   ┌─────────┐          ┌─────────┐          ┌─────────┐     │  │
│  │   │ ztunnel │          │ ztunnel │          │ ztunnel │     │  │
│  │   │ Node 1  │◄─HBONE──►│ Node 2  │◄─HBONE──►│ Node 3  │     │  │
│  │   └────┬────┘          └────┬────┘          └────┬────┘     │  │
│  │        │                    │                    │           │  │
│  └────────┼────────────────────┼────────────────────┼───────────┘  │
│           │                    │                    │               │
│  ┌────────┼────────────────────┼────────────────────┼───────────┐  │
│  │  L7    │   Optional: Waypoint Proxies (Envoy, pro Namespace) │  │
│  │        ▼                    ▼                                 │  │
│  │   ┌──────────┐        ┌──────────┐                           │  │
│  │   │ Waypoint │        │ Waypoint │   ← Nur bei L7-Bedarf    │  │
│  │   │ ns: app  │        │ ns: web  │     (AuthzPolicy, Retry) │  │
│  │   └──────────┘        └──────────┘                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Application Pods (KEIN Sidecar!)                            │   │
│  │   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐             │   │
│  │   │App A│  │App B│  │App C│  │App D│  │App E│             │   │
│  │   └─────┘  └─────┘  └─────┘  └─────┘  └─────┘             │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Zusammenfassung der Komponenten:**

| Helm Chart   | Funktion                                  | Typ        |
|-------------|-------------------------------------------|------------|
| `istio/base`  | CRDs (VirtualService, DestinationRule...) | Cluster    |
| `istio/istiod`| Control Plane (xDS, Cert-Mgmt)           | Cluster    |
| `istio/cni`   | CNI Plugin, Traffic-Redirect zu ztunnel  | DaemonSet  |
| `istio/ztunnel`| L4-Proxy, mTLS, HBONE-Tunnel            | DaemonSet  |
| Gateway API   | CRDs für Waypoint/Ingress-Konfiguration  | Cluster    |


---

### 2. Wie ArgoCD mit Istio Ambient zusammenspielt

#### GitOps-Workflow

```
┌──────────────┐    Push     ┌──────────────┐    Sync     ┌──────────────┐
│              │────────────►│              │────────────►│              │
│   Developer  │             │   Git Repo   │             │   ArgoCD     │
│              │◄────────────│              │◄────────────│   Controller │
│              │    PR/Review│  (Source of   │   Diff/     │              │
└──────────────┘             │   Truth)     │   Status    └──────┬───────┘
                             └──────────────┘                    │
                                                                 │ Helm
                                                                 │ Template
                                                                 │ + Apply
                                                                 ▼
                             ┌────────────────────────────────────────────┐
                             │          Kubernetes Cluster                │
                             │                                            │
                             │  ┌─────────┐ ┌────────┐ ┌───────────┐    │
                             │  │istio-   │ │istiod  │ │ztunnel    │    │
                             │  │base     │ │        │ │+ istio-cni│    │
                             │  │(CRDs)   │ │(CP)    │ │(DP)       │    │
                             │  └─────────┘ └────────┘ └───────────┘    │
                             │                                            │
                             │  ┌──────────────────────────────────┐     │
                             │  │  App-Namespace (ambient labeled) │     │
                             │  │  ┌──────┐ ┌──────┐ ┌──────────┐ │     │
                             │  │  │App A │ │App B │ │Waypoint  │ │     │
                             │  │  └──────┘ └──────┘ └──────────┘ │     │
                             │  └──────────────────────────────────┘     │
                             └────────────────────────────────────────────┘
```

#### Install-Reihenfolge (kritisch!)

ArgoCD deployed Helm Charts. Bei Istio Ambient gibt es **Abhängigkeiten**:

```
  Sync Wave 0        Sync Wave 1       Sync Wave 2        Sync Wave 3
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Gateway API │  │  istio-base  │  │   istiod     │  │  istio-cni   │
│  CRDs        │──►│  (CRDs)      │──►│  (profile:   │──►│  (profile:   │
│              │  │              │  │   ambient)   │  │   ambient)   │
└──────────────┘  └──────────────┘  └──────┬───────┘  └──────┬───────┘
                                           │                  │
                                           │                  ▼
                                           │          ┌──────────────┐
                                           └─────────►│   ztunnel    │
                                                      │              │
                                                      └──────────────┘
                                                              │
                                                              ▼
                                                      ┌──────────────┐
                                                      │  Apps +      │
                                                      │  Namespace   │
                                                      │  Labeling    │
                                                      └──────────────┘
```

**Reihenfolge:** Gateway API CRDs → istio-base → istiod → cni → ztunnel → Apps

---

### 3. Gotchas & Fallstricke

#### NetworkPolicy + HBONE Port 15008

Das ist der häufigste Fehler! Ambient Mode nutzt HBONE (HTTP/2 + mTLS) über **Port 15008**.
Bestehende NetworkPolicies blockieren diesen Port oft.

```
  PROBLEM:                                  LÖSUNG:
┌───────────────┐                        ┌───────────────┐
│  NetworkPolicy│                        │  NetworkPolicy│
│               │                        │               │
│  ingress:     │                        │  ingress:     │
│  - port: 8080 │  ← blockiert 15008!   │  - port: 8080 │
│               │                        │  - port: 15008│  ← HBONE erlauben!
└───────────────┘                        └───────────────┘
```

#### ArgoCD HA + Ambient Mode

ArgoCD HA nutzt Redis-HA mit Sentinel. Wenn der ArgoCD-Namespace ins Ambient Mesh
eingebunden wird, muss Port 15008 in **allen** ArgoCD-NetworkPolicies erlaubt werden.

**Empfehlung:** ArgoCD-Namespace **NICHT** ins Ambient Mesh einbinden – ArgoCD ist die Management-Ebene für Istio und muss unabhängig davon funktionieren.
ArgoCD verwaltet Istio – es muss nicht selbst im Mesh sein.


### 4. Initiales Ausrollen – Schritt für Schritt

```
Was Du tust (manuell)              Was ArgoCD tut (automatisch)
─────────────────────              ────────────────────────────

Schritt 1: Cluster + ArgoCD
  bereitstellen
         │
         ▼
Schritt 2: Git-Repo anlegen
  mit allen Application-YAMLs
         │
         ▼
Schritt 3: root-app.yaml
  kubectl apply (1× manuell!)
         │                         ArgoCD liest apps/ aus Git
         └────────────────────────►      │
                                         ├─► erstellt gateway-api-crds App
                                         ├─► erstellt istio-base App
                                         ├─► erstellt istiod App
                                         ├─► erstellt istio-cni App
                                         ├─► erstellt ztunnel App
                                         └─► erstellt bookinfo App
                                                     │
                                              Sync Waves steuern
                                              die Reihenfolge
                                                     │
                                                     ▼
                                              Istio Ambient läuft!

Ab jetzt: Änderungen NUR noch über Git-Commits.
```

#### Schritt 1: Voraussetzungen prüfen

```bash
## Kubernetes >= 1.28
kubectl version

## Helm >= 3.6 (nur zum Prüfen, ArgoCD nutzt Helm intern)
helm version
```

#### Schritt 2: ArgoCD installieren

```bash
## Falls noch kein ArgoCD im Cluster:
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## Warten bis alles läuft:
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

## Admin-Passwort holen:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

## Optional: Port-Forward für UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
## → https://localhost:8080  (User: admin)
```

#### Schritt 3: Git-Repo anlegen und befüllen

```bash
## Neues Repo erstellen (z.B. auf GitHub/GitLab)
mkdir istio-ambient-gitops && cd istio-ambient-gitops
git init

## Verzeichnisstruktur anlegen
mkdir -p apps/istio/values apps/bookinfo
```

Jetzt alle Application-YAMLs und Values-Dateien erstellen
(siehe Abschnitt 4.5 für den Inhalt jeder Datei):

```bash
## Dateien erstellen:
touch apps/istio/gateway-api-crds.yaml
touch apps/istio/istio-base-app.yaml
touch apps/istio/istiod-app.yaml
touch apps/istio/istio-cni-app.yaml
touch apps/istio/ztunnel-app.yaml
touch apps/istio/values/istiod-values.yaml
touch apps/bookinfo/bookinfo-app.yaml

## Alles committen und pushen:
git add -A
git commit -m "Initial Istio Ambient + ArgoCD setup"
git remote add origin https://github.com/DEIN-USER/istio-ambient-gitops.git
git push -u origin main
```

#### Schritt 4: root-app.yaml erstellen und anwenden

Diese Datei liegt NICHT im Git-Repo – sie ist der einzige manuelle Schritt:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

#### Schritt 5: Zuschauen und verifizieren

```bash
## ArgoCD Applications beobachten:
kubectl get applications -n argocd -w

## Erwartete Ausgabe nach ein paar Minuten:
## NAME                STATUS   HEALTH
## root                Synced   Healthy
## gateway-api-crds    Synced   Healthy
## istio-base          Synced   Healthy
## istiod              Synced   Healthy
## istio-cni           Synced   Healthy
## ztunnel             Synced   Healthy

## Istio-Pods prüfen:
kubectl get pods -n istio-system

## CRDs prüfen:
kubectl get crds | grep -E 'istio|gateway'
```

#### Ab jetzt: Alles über Git

```bash
## Beispiel: Neue App hinzufügen
vim apps/bookinfo/bookinfo-app.yaml
git add -A && git commit -m "Add bookinfo app"
git push
## → ArgoCD erkennt die Änderung und deployed automatisch

## Beispiel: Istio upgraden
## In apps/istio/*-app.yaml: targetRevision von 1.24.2 auf 1.25.0 ändern
git add -A && git commit -m "Upgrade Istio to 1.25.0"
git push
## → ArgoCD rolled die neuen Versionen aus
```

---

#### 4.4 Git-Repo Struktur

```
istio-ambient-gitops/
├── apps/
│   ├── istio/
│   │   ├── gateway-api-crds.yaml       # ArgoCD Application
│   │   ├── istio-base-app.yaml         # ArgoCD Application
│   │   ├── istiod-app.yaml             # ArgoCD Application
│   │   ├── istio-cni-app.yaml          # ArgoCD Application
│   │   ├── ztunnel-app.yaml            # ArgoCD Application
│   │   └── values/
│   │       ├── istiod-values.yaml
│   │       ├── cni-values.yaml
│   │       └── ztunnel-values.yaml
│   ├── bookinfo/
│   │   └── bookinfo-app.yaml           # ArgoCD Application
│   └── root-app.yaml                   # App-of-Apps
```

#### 4.5 ArgoCD Application Manifeste

##### Gateway API CRDs (Sync Wave -3)

```yaml
## apps/istio/gateway-api-crds.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-api-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/kubernetes-sigs/gateway-api.git
    targetRevision: v1.4.0
    path: config/crd/standard
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=false
```

##### istio-base (Sync Wave -2)

```yaml
## apps/istio/istio-base-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-base
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: base
    targetRevision: 1.24.2     # oder 1.29.x je nach Version
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true    # wichtig für CRDs!
```

##### istiod (Sync Wave -1)

```yaml
## apps/istio/istiod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istiod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: istiod
      targetRevision: 1.24.2
      helm:
        valueFiles:
          - $values/apps/istio/values/istiod-values.yaml
    - repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
```

**istiod-values.yaml:**
```yaml
## apps/istio/values/istiod-values.yaml
profile: ambient
pilot:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
meshConfig:
  accessLogFile: /dev/stdout
```

##### istio-cni (Sync Wave 0)

```yaml
## apps/istio/istio-cni-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-cni
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: cni
    targetRevision: 1.24.2
    helm:
      values: |
        profile: ambient
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system       # oder kube-system je nach Setup
  syncPolicy:
    automated:
      selfHeal: true
```

##### ztunnel (Sync Wave 1)

```yaml
## apps/istio/ztunnel-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ztunnel
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: ztunnel
    targetRevision: 1.24.2
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
```

##### Multi-Source Variante (alles in einer Application)

Alternativ kann man alle Charts in einer einzigen Application mit `sources` bündeln.
Das funktioniert, aber man verliert die Sync-Wave-Kontrolle:

```yaml
## Kompakte Variante – ACHTUNG: keine Reihenfolge-Garantie!
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-ambient
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  sources:
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: base
      targetRevision: 1.24.2
      helm:
        releaseName: istio-base
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: istiod
      targetRevision: 1.24.2
      helm:
        releaseName: istiod
        parameters:
          - name: profile
            value: ambient
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: cni
      targetRevision: 1.24.2
      helm:
        releaseName: istio-cni
        parameters:
          - name: profile
            value: ambient
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: ztunnel
      targetRevision: 1.24.2
      helm:
        releaseName: ztunnel
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

#### 4.6 App-of-Apps (Root Application)

```yaml
## apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
```

#### 4.7 Namespace ins Ambient Mesh einbinden

```yaml
## bookinfo-namespace.yaml (im Git-Repo!)
apiVersion: v1
kind: Namespace
metadata:
  name: bookinfo
  labels:
    istio.io/dataplane-mode: ambient    # ← Das ist der Schlüssel!
```

#### 4.8 Waypoint Proxy deployen (für L7-Features)

```yaml
## Nur nötig wenn AuthorizationPolicy, Retries, etc. auf L7 gebraucht werden
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-waypoint
  namespace: bookinfo
  labels:
    istio.io/waypoint-for: service      # oder "all" für den ganzen Namespace
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
```

#### 4.9 Verifizierung

```bash
## Istio-Komponenten prüfen
kubectl get pods -n istio-system
## Erwartung: istiod-xxx, ztunnel-xxx (pro Node), istio-cni-node-xxx (pro Node)

## CRDs prüfen
kubectl get crds | grep -E 'istio|gateway'

## Ambient-Enrollment prüfen
kubectl get namespace bookinfo --show-labels | grep dataplane-mode

## ztunnel-Logs: mTLS-Verbindungen sichtbar
kubectl logs -n istio-system -l app=ztunnel --tail=20

## Mesh-Status mit istioctl
istioctl ztunnel-config workloads
```

---

### 5. Checkliste: Was muss ich beachten?

```
 ✓  Gateway API CRDs VOR Istio installieren (Sync Wave!)
 ✓  Helm Charts in korrekter Reihenfolge: base → istiod → cni → ztunnel
 ✓  ServerSideApply=true für CRD-haltige Charts setzen
 ✓  profile: ambient bei istiod und cni setzen
 ✓  NetworkPolicies: Port 15008 (HBONE) freigeben
 ✓  ArgoCD-Namespace NICHT ins Ambient Mesh (Management-Ebene bleibt unabhängig)
 ✓  Namespace-Label: istio.io/dataplane-mode=ambient
 ✓  Waypoint nur deployen, wenn L7-Features nötig sind
 ✓  Alle Istio-Versionen einheitlich halten (base, istiod, cni, ztunnel)
 ✓  Bei Upgrades: targetRevision in Git ändern → ArgoCD synced automatisch
```

---

### 6. Upgrade-Workflow

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│ 1. Git Branch  │     │ 2. PR Review   │     │ 3. Merge       │
│                │     │                │     │                │
│ targetRevision │────►│ Diff prüfen:   │────►│ ArgoCD synced  │
│ 1.24.2 → 1.25.0│    │ - Release Notes│     │ automatisch    │
│                │     │ - Breaking Ch. │     │                │
└────────────────┘     └────────────────┘     └────┬───────────┘
                                                    │
                                        ┌───────────┼───────────┐
                                        │           │           │
                                        ▼           ▼           ▼
                                    base CRDs    istiod     ztunnel
                                    (update)    (rollout)  (rollout)
```

**Wichtig:** ztunnel ist ein DaemonSet – beim Upgrade gibt es einen kurzen
Traffic-Umbruch pro Node. In Produktion: `maxUnavailable: 1` in den Values setzen.

## Misc

### Grenze bei der Header-Größe Istio vs. Nginx


### Problem

Keycloak erzeugt große HTTP-Header (JWTs, Set-Cookie mit Access/ID/Refresh Token). Sowohl nginx als auch Istio/Envoy haben Default-Limits, die dabei überschritten werden können.

---

### Envoy/Istio Defaults

| Parameter                 | Default  | Maximum  |
|---------------------------|----------|----------|
| `max_request_headers_kb`  | 60 KiB*  | **96 KiB** (hartes Limit) |
| `max_headers_count`       | 100      | frei konfigurierbar |

> **Achtung:** In älteren Istio-Versionen (< 1.8) lag der tatsächliche Default im Code bei ~29 KiB, nicht bei den dokumentierten 60 KiB.

**Fehlermeldung bei Überschreitung:** `431 Request Header Fields Too Large`

### nginx Defaults

| Parameter                    | Default     |
|------------------------------|-------------|
| `large_client_header_buffers`| 4 × 8 KB   |
| `proxy_buffer_size`          | 4k / 8k     |
| `proxy_buffers`              | 8 × 4k/8k  |

**Fehlermeldungen bei Überschreitung:**

- **400 Bad Request** — Request-Header des Clients zu groß (`large_client_header_buffers`)
- **502 Bad Gateway** — Response-Header vom Upstream (Keycloak) zu groß (`proxy_buffer_size`) — der Klassiker mit Keycloak

---

### Fix: Istio/Envoy (EnvoyFilter)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: max-request-headers
  namespace: istio-system
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: ANY
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: MERGE
      value:
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
          max_request_headers_kb: 96
```

**Verifizierung:**

```bash
istioctl proxy-config listener <pod-name> -o json | grep max_request_headers
```

#### Bekannte Probleme

- Das **Verringern** des Limits funktioniert zuverlässig, das **Erhöhen** über den Default hinaus greift manchmal nicht (gemeldetes Istio-Issue)
- Werte > 96 werden ignoriert — **128 ist ungültig!**
- Der `@type` muss zur Istio-Version passen:
  - Istio ≤ 1.6: `...http_connection_manager.v2.HttpConnectionManager`
  - Istio ≥ 1.8: `...http_connection_manager.v3.HttpConnectionManager`

---

### Fix: nginx

```nginx
## Response-Header vom Upstream (Keycloak) — behebt 502
proxy_buffer_size       128k;
proxy_buffers           4 256k;
proxy_busy_buffers_size 256k;

## Request-Header vom Client — behebt 400
large_client_header_buffers 4 32k;
```

---

### Checkliste: Weitere Header-Limits prüfen

Nicht nur der Proxy, auch die **Anwendung selbst** kann Header ablehnen:

| Framework / Server | Default Max Header |
|--------------------|--------------------|
| Spring Boot        | 8 KiB              |
| Gunicorn           | 8 KiB              |
| Tomcat             | 8 KiB              |
| Node.js (http)     | 16 KiB             |

Falls nach dem EnvoyFilter-Fix weiterhin Fehler auftreten, liegt es oft am App-Container, nicht an Envoy.

### Istio als API Gateway


### Was sind Dinge, die istio nicht unterstützt

**Features, die Istio NICHT bietet:**

- **Request/Response Body Transformation** — kein Umschreiben von JSON-Bodys, Feld-Mapping, XML↔JSON-Konvertierung (nur Header-Manipulation möglich)
- **OpenAPI-Spec-Validierung** — keine automatische Validierung eingehender Requests gegen ein Schema
- **API Key Management** — kein Ausstellen, Rotieren, Widerrufen von API Keys
- **Developer Portal** — keine Self-Service-Oberfläche für API-Consumer
- **Quota Management pro Consumer** — Rate Limiting ja, aber keine feingranularen Quotas pro API-Key/Consumer-Plan
- **API Versionierungs-Management** — Routing auf v1/v2 geht, aber kein automatisches Deprecation/Sunset-Handling
- **Caching** — kein Response-Caching am Gateway
- **GraphQL-Support** — kein nativer GraphQL-zu-REST-Proxy oder Schema-Stitching
- **Monetarisierung / Billing** — keine Usage-Tracking-Integration für Abrechnung

### Was sind Features, die istio unterstützt 

**Was oft verwechselt wird — das kann Istio durchaus:**
- Rate Limiting (via WASM oder EnvoyFilter)
- JWT-Validierung / OIDC
- Header-basiertes Routing
- mTLS / AuthorizationPolicy

**Kernaussage:** Istio fehlt vor allem alles rund um **Body-Manipulation, Schema-Validierung und API-als-Produkt-Features**. Auf der Traffic- und Security-Ebene ist es sehr vollständig.

### Mehrere Namepaces bei allowedRoutes bei Gateway - Objekt / Gateway API


** Mit Label-Selector (empfohlen)**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
```

Dann die gewünschten Namespaces labeln:

```bash
kubectl label ns app1 gateway-access=true
kubectl label ns app2 gateway-access=true
```

```

Es gibt **keine** Möglichkeit, einzelne Namespaces als Liste anzugeben — die Gateway API bietet nur `Same`, `All` oder `Selector`. Der Label-Selector ist der Weg, wenn du gezielt bestimmte Namespaces freischalten willst.

### Global Ratelimiting, je nach Kunde


Komplettes Setup für per-Kunde Rate Limiting im Istio Service Mesh.
Kunden authentifizieren sich per JWT, der `customer_id`-Claim wird als
Envoy-Descriptor an den externen Rate Limit Service gesendet. Monitoring
über StatsD → statsd-exporter → Prometheus/VictoriaMetrics → Grafana.

---

### Architektur

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

#### Begriffe

- **Descriptor**: Key-Value-Paar (z.B. `customer_id: "customer-a"`), das Envoy per gRPC an den Rate Limit Service schickt. Wird über `actions` im EnvoyFilter definiert. Mögliche Quellen: JWT Claims, HTTP Header, Client-IP, statische Werte.
- **Cluster-Patch**: Registriert den Rate Limit Service als Envoy-Cluster im Gateway mit einem kontrollierten Namen (`rate_limit_cluster`). Auch wenn der RLS im Mesh läuft und Istio automatisch einen Cluster anlegt, ist der explizite Patch der sauberere Weg — du kontrollierst den Cluster-Namen, der im HTTP Filter referenziert wird.
- **Domain**: Logischer Namespace im RLS, um Configs verschiedener Anwendungen zu trennen. Muss in EnvoyFilter und ConfigMap identisch sein.

---

### Komponenten

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

### 1. Namespace (mit Sidecar Injection)

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

### 2. Redis

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

### 3. Rate Limit Config (pro Kunde)

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

### 4. StatsD-Exporter Mapping Config

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

### 5. Rate Limit Service + StatsD-Exporter Sidecar

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

### 6. Monitoring Scrape Config

> **Hinweis**: Das Prometheus aus dem Istio Demo-Stack (`istio/samples/addons`)
> ist ein **Plain Prometheus ohne Operator**. CRDs wie `PodMonitor` und
> `PrometheusRule` existieren dort nicht.

#### Variante A: Plain Prometheus / Istio Demo (Standard)

Kein extra Manifest nötig. Das Deployment in Schritt 5 hat bereits die
Annotations gesetzt, die Plain Prometheus automatisch scraped:

```yaml
## Bereits im Deployment (Schritt 5) enthalten:
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9102"
  prometheus.io/path: "/metrics"
```

#### Variante B: Prometheus Operator / VictoriaMetrics Operator

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

### 7. JWT Authentication (am Gateway)

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

### 8. AuthorizationPolicy (JWT erzwingen)

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

### 9. EnvoyFilter: Rate Limit am Gateway

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

### 10. PeerAuthentication (nötig weil ratelimit im Mesh)

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

### 11. Alerting

> **Hinweis**: Das Prometheus aus dem Istio Demo-Stack (`istio/samples/addons`)
> hat keinen Operator. `PrometheusRule` CRDs werden dort nicht unterstützt.

#### Variante A: Plain Prometheus / Istio Demo

Die Rules direkt als ConfigMap einbinden:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-ratelimit-rules
  namespace: istio-system    # oder wo dein Prometheus läuft
data:
  ratelimit_rules.yml: |
    groups:
    - name: ratelimit
      rules:
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

      - alert: RateLimitServiceDown
        expr: up{job="ratelimit"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Rate Limit Service ist nicht erreichbar"

      - alert: RateLimitConfigError
        expr: increase(ratelimit_service_config_load_error[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Rate Limit Config konnte nicht geladen werden"
```

Die ConfigMap muss als Volume in den Prometheus-Pod gemountet und in `prometheus.yml` referenziert werden:

```yaml
rule_files:
- /etc/prometheus/ratelimit_rules.yml
```

#### Variante B: Prometheus Operator

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ratelimit-alerts
  namespace: ratelimit
spec:
  groups:
  - name: ratelimit
    rules:
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

    - alert: RateLimitServiceDown
      expr: up{job="ratelimit"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Rate Limit Service ist nicht erreichbar"

    - alert: RateLimitConfigError
      expr: increase(ratelimit_service_config_load_error[5m]) > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Rate Limit Config konnte nicht geladen werden"
```


---

### Deployment-Reihenfolge

```bash
## 1. Namespace
kubectl apply -f 01-namespace.yaml

## 2. Redis
kubectl apply -f 02-redis.yaml
kubectl -n ratelimit wait --for=condition=ready pod -l app=redis --timeout=60s

## 3. Configs (ratelimit + statsd mapping)
kubectl apply -f 03-ratelimit-config.yaml
kubectl apply -f 04-statsd-config.yaml

## 4. Rate Limit Service + StatsD Exporter
kubectl apply -f 05-ratelimit-deployment.yaml
kubectl -n ratelimit wait --for=condition=ready pod -l app=ratelimit --timeout=60s

## 5. Monitoring (nur bei Prometheus Operator, sonst überspringen - Annotations reichen)
## kubectl apply -f 06-podmonitor.yaml

## 6. PeerAuthentication (nötig weil ratelimit im Mesh)
kubectl apply -f 07-peer-authentication.yaml

## 7. JWT Auth + Policy
kubectl apply -f 08-request-authentication.yaml
kubectl apply -f 09-authorization-policy.yaml

## 8. EnvoyFilter (zuletzt!)
kubectl apply -f 10-envoyfilter-ratelimit.yaml

## 9. Alerting (PrometheusRule bei Operator, sonst ConfigMap - siehe Schritt 11)
## kubectl apply -f 11-prometheus-rules.yaml
```

---

### Testen

```bash
## Token für customer-a generieren
TOKEN_A="eyJhbGciOiJS..."  # JWT mit claim: customer_id: "customer-a"

## Einzelner Request - sollte 200 zurückgeben
curl -v -H "Authorization: Bearer $TOKEN_A" https://api.example.com/endpoint

## Ohne Token - sollte 403 zurückgeben (AuthorizationPolicy)
curl -v https://api.example.com/endpoint

## Last generieren - Rate Limit testen
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN_A" \
    https://api.example.com/endpoint
done
## Erwartete Response bei Rate Limit:
## HTTP/1.1 429 Too Many Requests
## x-envoy-ratelimited: true

## Metriken direkt vom statsd-exporter prüfen
kubectl -n ratelimit port-forward deploy/ratelimit 9102:9102 &
curl -s http://localhost:9102/metrics | grep ratelimit_service_rate_limit
## Erwarteter Output:
## ratelimit_service_rate_limit_total_hits_count{domain="my-api",key1="customer_id_customer-a"} 50
## ratelimit_service_rate_limit_within_limit_count{domain="my-api",key1="customer_id_customer-a"} 40
## ratelimit_service_rate_limit_over_limit_count{domain="my-api",key1="customer_id_customer-a"} 10

## Redis Counter direkt prüfen
kubectl -n ratelimit exec -it deploy/redis -- redis-cli KEYS '*'
```

---

### Grafana / Prometheus Queries

#### Requests pro Kunde (Rate/Minute)

```promql
rate(ratelimit_service_rate_limit_total_hits_count{domain="my-api"}[1m]) * 60
```

#### Over-Limit Requests pro Kunde

```promql
rate(ratelimit_service_rate_limit_over_limit_count{domain="my-api"}[1m]) * 60
```

#### Near-Limit Warnungen (>80% verbraucht)

```promql
rate(ratelimit_service_rate_limit_near_limit_count{domain="my-api"}[1m]) > 0
```

#### Prozent vom Limit verbraucht

Die Metrik weiß nicht, wie hoch das konfigurierte Limit ist.
Statische Zuordnung als Recording Rule oder Grafana-Variable nötig:

```promql
## customer-a: 1000 req/min
rate(ratelimit_service_rate_limit_total_hits_count{key1="customer_id_customer-a"}[1m]) * 60 / 1000 * 100

## customer-b: 100 req/min
rate(ratelimit_service_rate_limit_total_hits_count{key1="customer_id_customer-b"}[1m]) * 60 / 100 * 100
```

#### Config Load Status (Healthcheck)

```promql
ratelimit_service_config_load_success   # Sollte > 0 sein
ratelimit_service_config_load_error     # Sollte 0 sein
```

---

### Debugging

```bash
## Rate Limit Service Logs
kubectl -n ratelimit logs -l app=ratelimit -c ratelimit -f

## StatsD Exporter Logs
kubectl -n ratelimit logs -l app=ratelimit -c statsd-exporter -f

## Gateway: Wurde der Cluster-Patch applied?
istioctl proxy-config cluster -n istio-system deploy/istio-ingressgateway | grep rate

## Gateway: Wurde der HTTP Filter applied?
istioctl proxy-config listener -n istio-system deploy/istio-ingressgateway -o json | grep ratelimit

## Gateway: Route Config mit Rate Limit Actions?
istioctl proxy-config route -n istio-system deploy/istio-ingressgateway -o json | grep rate

## Redis Counter prüfen
kubectl -n ratelimit exec -it deploy/redis -- redis-cli KEYS '*'
```

#### Häufige Fehler

| Symptom | Ursache | Lösung |
|---------|---------|--------|
| `unknown cluster 'rate_limit_cluster'` | Cluster-Patch fehlt oder falsch | EnvoyFilter Cluster-Patch prüfen |
| Rate Limit greift nicht | Domain in EnvoyFilter ≠ ConfigMap | `domain: my-api` überall identisch |
| 500 statt 429 | RLS nicht erreichbar + `failure_mode_deny: true` | RLS Logs prüfen, auf `false` setzen |
| Metriken leer | `USE_STATSD=false` oder statsd-exporter nicht da | Env-Var + Sidecar prüfen |
| Metriken ohne Kunden-Labels | Mapping-Config fehlt oder falsch | statsd-config ConfigMap prüfen |

---

### Hinweise für Produktion

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

### zu istio beitragen



**Der typische Weg:**

1. **Einstieg über Docs/UX** — Am einfachsten ist der Einstieg über Issues mit den Labels "User Experience" und "Documentation". Auf `istio/istio.io` gibt es kuratierte "good first issues", die speziell für Erstbeitragende ausgewählt sind.

2. **CLA unterschreiben** — Bevor PRs akzeptiert werden, muss ein Contributor License Agreement (CLA) bei der CNCF/Linux Foundation unterzeichnet werden.

3. **Working Groups** — Für Feature-Beiträge muss man zuerst die Idee mit der zuständigen Working Group auf Slack diskutieren, dann ein GitHub Issue erstellen, und bei größeren Features ein Design-Dokument über Google Docs einreichen.

4. **Release Manager Rotation** — Wer tiefer einsteigen will, kann der Release-Manager-Rotation beitreten — zwei Releases als Shadow, dann selbst Primary.


### Ref:

  * https://github.com/istio/community/blob/master/CONTRIBUTING.md#working-groups
