# Architektur & Komponenten von Istio

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
```mermaid
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
```mermaid
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
