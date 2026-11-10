# Istio & Kubernetes 

## Agenda

  1. Kubernetes - Grundlagen
     * [Anatomie einer Webanwendung mit der Gateway API (ohne Istio)](kubernetes-grundlagen/01-anatomie-webanwendung-gateway-api.md)

  1. Grundlagen ServiceMesh & Istio
     * [Einführung in Istio & Service Mesh-Architekturen](/istio/overview/01-introduction.md)
     * [Warum ein Service Mesh?](istio/overview/02-warum-servicemesh.md)
     * [Herausforderung & Vorteile](istio/overview/03-herausforderungen-vorteile.md)
     * [Architektur & Komponenten von Istio (Ambient: istiod, ztunnel, Waypoint, istio-cni)](istio/overview/04-architektur-komponenten.md)
     * [Waypoint: was bedeutet --for service, workload und all?](istio/overview/10-waypoint-for-service-oder-workload.md)
     * [Istio Ingress Gateway vs. Kubernetes Gateway API](istio/overview/06-ingress-istio-gateway-vs-kubernetes-gateway-api.md)

  1. istioctl
     * [istioctl cheatsheet zum Debuggen](/istio/istioctl-cheatsheet.md)

  1. Installation von istio (ambient mode)
     * [istio-installation mit istioctl - ambient (Installation per istioctl, um ein Gefühl dafür zu bekommen)](istio/installation/ambient/03-install-with-istioctl-ambient.md)
     * [Bookinfo demo in ambient ausrollen](istio/installation/ambient/04-install-demo-app.md) 
     * [istio (ambient) deinstallieren - nur als Referenz](istio/installation/ambient/99-deinstallation-ambient.md)

  1. Installation auf OCP/Azure (ambient mode)
     * [Istio Ambient auf OpenShift (OCP): OSSM 3 per Operator, Unterschiede und Einschränkungen](istio/installation/openshift/01-ossm-ambient-ocp.md)
     * [Rancher auf Azure im Hub-and-Spoke-Netz: Verbindungen, Firewall, Istio Ambient auf den Downstream-Clustern](istio/installation/openshift/03-hub-spoke-rancher-azure.md)

  1. Steuerung des Netzwerkverhaltens in Istio
     * [Gateway API als Nachfolger von IngressGateway (Istio), VirtualService und DestinationRule](istio/overview/08-virtualservice-destinationrule-gateway-api.md)
     * [Virtual Services, Destination Rules](istio/overview/09-funktionsweise-virtualservice-destinationrule.md)

  1. Übungen Steuerung des Netzwerkverhaltens (Gateway API)
     * [Übung: Header-basiertes Routing](istio/exercises/02-exercise-request-routing.md)
     * [Übung: Traffic-Shifting / Load-Balancing](istio/exercises/04-traffic-shifting.md)
     * [Pfadbasiertes Routing](istio/exercises/13-pfadbasiertes-routing-gateway-api.md)
     * [Übung: LoadBalancing (LEAST_CONN) - Regeln](istio/exercises/15-loadbalancing-least-conn-gateway-api.md)
     * [Übung: Bookinfo mit HTTPS und Let's Encrypt (cert-manager, Gateway API)](istio/exercises/17-https-letsencrypt-bookinfo.md)

  1. Service Resilience & Fehlertoleranz (Gateway API)
     * [Circuit Breaker](istio/exercises/06-circuit-breaker.md) 
     * [Circuit Breaker - Gateway API (Ambient + Waypoint)](istio/exercises/gateway-api/06-circuit-breaker-gateway-api.md)
     * [Chaos Engineering mit Istio](https://istio.io/latest/docs/examples/microservices-istio/production-testing/)

  1. Sichere Service-Kommunikation mit istio (Sicherheit)
     * [Sicherheit & Zero Trust mit Istio](/istio/overview/zero-trust.md)
     * [Was ist in istio deep-defense (defense in depth) ?](/istio/overview/defense-in-depth.md)   
     * [Überblick 3 Säulen -  Sichere Service-Kommunikation](istio/overview/security/01-secure-service-communication.md)
     * [Übung Säule 1 (Sidecar): mTLS prüfen und aktivieren/deaktivieren](istio/overview/security/02-check-mtls-enable-disable.md)
     * [Übung Säule 2 + 3: Authentication und Authorization mit JWT und RBAC](istio/exercises/12-jwt-rbac.md)

  1. Sichere Service-Kommunikation mit istio im Ambient-Mode (mTLS)
     * [Übung Säule 1 (Ambient): mTLS prüfen und aktivieren/deaktivieren (ztunnel, HBONE, PeerAuthentication)](istio/overview/security/02-check-mtls-enable-disable-ambient.md)

  1. Sicherheit, Fehlertoleranz & Observability    
     * Istio Authorization Policies (ingress/egress Security)
     * [Übung (Ambient): Zero Trust für bookinfo Schritt für Schritt (ztunnel Layer 4, Waypoint Layer 7)](istio/exercises/19-zerotrust-bookinfo-ambient.md)
     * [Übung (Sidecar): ZeroTrust für bookinfo Namespace bookinfo einrichten](istio/exercises/16-zerotrust-bookinfo-einrichten.md)
     * [Übung (Ambient): Egress-Gateway mit Waypoint - externe Ziele steuern, TLS-Origination, Weg per NetworkPolicy erzwingen](istio/exercises/20-egress-gateway-ambient.md)

  1. Debugging
     * [Debugging mit debug/run pod](/istio/debug/01-debug-pod.md)
     * [Übung: Fehlersuche im Ambient-Mesh - ztunnel-Log und Waypoint-Log](istio/exercises/18-ztunnel-log-debugging.md)

  1. Monitoring, Logging & Observability
     * [Installation Jaeger](istio/installation/addons/jaeger.md)
     * [Distributed Tracing mit Jaeger](istio/exercises/09-tracing-jaeger.md)
     * Metriken & Dashboards mit Prometheus & Grafana
     * [Installation Prometheus Addon with Ingress](istio/installation/addons/prometheus.md)
     * [Metriken mit Prometheus auswerten](istio/exercises/10-prometheus.md) 
     * [Installation Grafana Addon with Ingress](istio/installation/addons/grafana.md)
     * [Grafana Dashboards für istio](istio/exercises/11-grafana.md)
     * [Sinnvolle istio Metriken und Dashboards - Teil 1](istio/observeability/istio-observability-in-grafana.md)
     * [Sinnvolle istio Metriken und Dashboards - Teil 2](https://github.com/jmetzger/training-istio-kubernetes/blob/main/istio/observeability/istio-observeablity-in-prometheus-2.md)
     * [Installation Kiali // Installation](istio/installation/addons/kiali.md)
     * [Visualisierung mit Kiali](istio/exercises/05-kiali-traffic.md)
     * [Analyse & Debugging von Service-Mesh-Daten](istio/strategie/mesh-daten-analysieren-und-debuggen.md)

  1. Vergleich Sidecar und Ambient
     * [Istio Proxy-Konzepte](istio/overview/07-istio-proxy-concepts-sidecar.md)
     * [Istio Sidecar-Injection](istio/exercises/01-exercise-injection.md)
     * Sidecar-Overhead & Ressourcenoptimierung
     * [Vergleich ohne istio, istio sidecar, istio ambient](/istio/overview/performance-comparison-baseline-sidecar-ambient.md)
     * [Was Ambient (noch) nicht kann, was Sidecar kann (Stand 10.11.2026)](istio/overview/11-ambient-grenzen-gegenueber-sidecar.md)
     * Ressourcenverbrauch messen: Sidecar-Modus gegen Ambient-Modus mit denselben Pods
     * [Bookinfo von Sidecar auf Ambient umstellen](istio/installation/ambient/04-upgrade-bookinfo-sidecar-2-ambient.md)
     * [Canary Update von istiod mit tags und revisions](/istio/installation/ambient/05-istio-canary-upgrade-revisions.md)

  1. Multi-Cluster (nur Theorie)
     * Multi-Cluster- & Hybrid-Umgebungen mit Istio
     * Istio Federation & Cross-Cluster Traffic

  1. Erweiterbarkeit von Istio (nur Theorie)
     * WebAssembly (Wasm) für Istio-Erweiterungen
     * [Wo läuft WASM (WebAssembly) im Rahmen von istio ?](istio/wasm/plugins/where-is-the-wasm-runtime.md)

  1. Automatisierung mit GitOps & ArgoCD
     * [Was ist ArgoCD](/istio/argocd/argocd-istio-bookinfo.md)
     * [istio ambient und bookinfo mit argocd ausrollen](/istio/argocd/argocd-istio-bookinfo.md)

## Backlog

  1. Installation von istio (ambient mode) mit helm
     * [istio-installation with helm - ambient](istio/installation/ambient/03-install-with-helm.md)

  1. Setup Cluster (Self-Service)
     * [Self-Service Cluster ausrollen](/istio/training-stack/install.md)
     * [Self-Service Cluster destroy](/istio/training-stack/uninstall.md)

  1. istio sidecar (Grundlagen)
     * [Systemanforderungen & Kubernetes-Cluster-Vorbereitung](istio/requirements/overview.md)

  1. Installation/Konfiguration/Deinstallation von istio (sidecar - Installation mit istioctl)
     * [Istio-Installation mit istioctl und der IstioOperator - Resource](istio/installation/03-install-with-istioctl-with-demo-profile.md)
     * [Installations-Config-Profile](istio/installation/02-installation-config-profiles.md)
     * [Wie ändere ich die Config/Installation von istio - Beispiel egressGateway](istio/installation/04-change-installation-with-istioctl-and-istioOperator.md)
     * [Deinstallation von Istio mit istioctl](istio/installation/05-uninstall.md)

  1. Installation von istio (sidecar) mit helm-charts
     * [Systemanforderungen & Kubernetes-Cluster-Vorbereitung](istio/requirements/overview.md)
     * [Istio-Installation mit helm charts](istio/installation/03a-install-with-helm-with-demo-profile.md)
     * [istioctl als debugging - tool installieren](istio/installation/03b-install-istioctl.md)

  1. Demo-App (Gateway API, sidecar)
     * [Istio demo-app *bookinfo* installieren (Gateway API - Variante)](istio/installation/04-install-demo-app-bookinfo.md)

  1. CNI-Plugin istio
     * [Vorteile und Installation mit helm](istio/installation/06-istio-cni-plugin-helm.md)

  1. Service Resilience & Fehlertoleranz (nur VirtualService)
     * [Bemerkungen zu nur VirtualService und nicht Gateway API - httproute](istio/overview/virtualservice-only-features-why.md)
     * [Retries VirtualService only !](istio/exercises/07-retries.md)
     * [Fehlerinjektion (z.B. 500er) VirtualService only !](istio/exercises/03-fault-injection.md)

  1. Service Resilience & Fehlertoleranz (nur Sidecar, EnvoyFilter)
     * [Local Rate Limit mit EnvoyFilter - nur Sidecar !](istio/exercises/08-local-rate-limit.md)

  1. Demo-App (istio api)
     * [Istio demo-app *bookinfo* installieren (istio API - Variante)](istio/installation/istio-api/04-install-demo-app-bookinfo.md)

  1. Übungen Steuerung des Netzwerkverhaltens (istio api)
     * [Übung: Header-basiertes Routing - istio api](istio/exercises/virtualservice/02-exercise-request-routing.md)

  1. Skalierung, Erweiterbarkeit & Performance-Optimierung
     * [Skalierung von Istio - Möglichkeiten/Tipps&Tricks](istio/skalierung-performance/skalierung.md)
     * [Performance-Optimierung](istio/skalierung-performance/performance.md)

  1. Erweiterte Routing-Techniken & Traffic-Optimierung (Istio API)
     * [Canary Releases & Progressive Deployments / Traffic - Shifting](istio/exercises/virtualservice/04-traffic-shift.md)
     * A/B-Tests & Traffic Mirroring
     * Blue-Green- und Canary-Deployments mit Istio

  1. Erweiterbarkeit von Istio (WebAssembly, Übung)
     * [Eigene Istio-Erweiterungen mit WebAssembly schreiben (RateLimit-Plugin)](istio/exercises/16-create-wasm-plugin-ratelimit.md)

  1. Misc
     * [Grenze bei der Header-Größe Istio vs. Nginx](istio/misc/header-size-istio-vs-nginx.md)
     * [Istio als API Gateway](istio/misc/istio-als-api-gateway.md)
     * [Mehrere Namepaces bei allowedRoutes bei Gateway - Objekt / Gateway API](istio/misc/mehrere-allowedRoutes-in-gateway-api-gateway.md)
     * [Global Ratelimiting, je nach Kunde](istio/misc/global-rate-limiting-pro-kunde.md)
     * [zu istio beitragen](istio/misc/zu-istio-beitragen.md)
