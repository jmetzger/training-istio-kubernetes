# Einführung in Istio & Service Mesh-Architekturen

**Was ist ein Service Mesh?**
- Dedizierte Infrastrukturschicht für Service-zu-Service-Kommunikation
- Transparente Zwischenschicht ohne Code-Änderungen
- Zentrale Steuerung von Retry, Timeout, Verschlüsselung, Monitoring

**Istio im Überblick:**
- Open-Source Service Mesh (Google, IBM, Lyft)
- Nutzt Envoy-Proxies als Sidecars
- Fängt gesamten Netzwerkverkehr ab




## Source / Mermaid 

```mermaid
graph LR
    subgraph "Ohne Service Mesh"
    A1[Service A] -.-> B1[Service B]
    A1 -.-> C1[Service C]
    B1 -.-> C1
    B1 -.-> D1[Service D]
    C1 -.-> D1
    end
    
    subgraph "Mit Service Mesh"
    A2[Service A] --> PA[Envoy Proxy]
    B2[Service B] --> PB[Envoy Proxy]
    C2[Service C] --> PC[Envoy Proxy]
    D2[Service D] --> PD[Envoy Proxy]
    PA <--> PB
    PA <--> PC
    PB <--> PC
    PB <--> PD
    PC <--> PD
    end
    
    style PA fill:#4285f4
    style PB fill:#4285f4
    style PC fill:#4285f4
    style PD fill:#4285f4
```
