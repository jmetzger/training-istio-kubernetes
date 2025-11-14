# Einführung in Istio & Service Mesh-Architekturen

## Was ist ein Service Mesh?

Ein Service Mesh ist eine dedizierte Infrastruktur-Layer für die Service-to-Service-Kommunikation in Microservices-Architekturen. Es entkoppelt Netzwerk-Logik (Routing, Security, Observability) vom Application-Code.

**Kernprinzip:** Sidecar-Proxies neben jedem Service übernehmen die komplette Netzwerk-Kommunikation.

## Welche Probleme löst ein Service Mesh?

- **Traffic Management:** Load Balancing, Canary Deployments, A/B Testing, Circuit Breaking
- **Security:** mTLS zwischen Services, Authorization Policies
- **Observability:** Distributed Tracing, Metriken, Logs ohne Code-Änderungen
- **Resilienz:** Retries, Timeouts, Fault Injection

Ohne Service Mesh müsste jeder Service diese Features selbst implementieren → Code-Duplikation, unterschiedliche Implementierungen, schwer wartbar.

## Was ist Istio?

Istio ist ein Open-Source Service Mesh für Kubernetes (und andere Plattformen). Entwickelt von Google, IBM und Lyft.

**Istio nutzt Envoy** als Sidecar-Proxy (performanter C++ Proxy von Lyft).

## Istio Architektur

### Data Plane
- **Envoy Proxies** als Sidecars in jedem Pod
- Intercepten gesamten Netzwerk-Traffic (inbound/outbound)
- Setzen Routing-Rules, Security Policies, Telemetrie um

### Control Plane
- **Istiod:** Zentrale Komponente (seit Istio 1.5)
  - Pilot: Service Discovery & Traffic Management
  - Citadel: Certificate Management (mTLS)
  - Galley: Configuration Management

**Flow:** Istiod pusht Konfiguration zu allen Envoy-Proxies.

## Hauptfunktionen

### 1. Traffic Management
- Virtual Services: Routing-Rules (z.B. 90% v1, 10% v2)
- Destination Rules: Load Balancing, Circuit Breaking
- Gateways: Ingress/Egress Traffic

### 2. Security
- Automatisches mTLS zwischen Services
- Authorization Policies (wer darf mit wem sprechen?)
- JWT Authentication

### 3. Observability
- Automatische Metriken (Latency, Error Rate, Traffic)
- Distributed Tracing (Jaeger)
- Access Logs

## Quick Facts

- **Installation:** Helm Chart, Helm Operator oder istioctl
- **Overhead:** ~1-2ms Latenz durch Sidecar, ~0.5 vCPU + 50MB RAM pro Proxy
- **Alternativen:** Linkerd, Consul Connect, AWS App Mesh
- **Istio seit 2017**, produktionsreif seit Version 1.0 (2018)

## Warum Istio?

✅ Standardisiertes Traffic Management ohne Code-Änderungen  
✅ Zero-Trust Security out-of-the-box  
✅ Umfassende Observability  
✅ Große Community, Enterprise-Support verfügbar  

⚠️ Komplexität steigt, Learning Curve  
⚠️ Performance Overhead durch Sidecars
