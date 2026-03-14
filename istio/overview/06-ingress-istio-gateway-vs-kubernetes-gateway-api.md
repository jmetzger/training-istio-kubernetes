# Ingress Istio-Gateway vs. Kubernetes Gateway API 

## Istio Ingress-Gateway-Flow (bookinfo/productpage) 

![Flow](/images/screenshot_20260314_142715.png)

## Gateway API -> Gateway Flow (bookinfo/productpage)

<img width="1024" height="703" alt="image" src="https://github.com/user-attachments/assets/a8be8bb3-a761-4fc2-b482-9a6798b49eda" />

   * Achtung:  VirtualService und Destination werden teilweise immer noch benötigt, auch wenn man mit der Gateway API arbeitet,
     um bestimmte Feature abzubilden  

## Gateway II 

```mermaid
graph TD
    Client[Client/Browser]
    GW[Gateway<br/>gateway.networking.k8s.io]
    GWC[GatewayClass<br/>istio / envoy / traefik]
    HR[HTTPRoute<br/>routing rules]
    REFS["backendRef: reviews-v1, weight: 80<br/>backendRef: reviews-v2, weight: 20"]
    SvcV1[Service reviews-v1<br/>selector: version=v1]
    SvcV2[Service reviews-v2<br/>selector: version=v2]
    NOTE["⚡ Keine DestinationRule nötig —<br/>Gewichtung direkt in HTTPRoute backendRefs"]

    Client -->|HTTP/HTTPS| GW
    GW -.->|instance of| GWC
    HR -.->|attached to| GW
    HR --> REFS
    REFS -->|"80%"| SvcV1
    REFS -->|"20%"| SvcV2

    style GWC fill:#eaf3de,stroke:#3b6d11
    style GW fill:#eaf3de,stroke:#3b6d11
    style HR fill:#eaf3de,stroke:#3b6d11
    style REFS fill:#f1efe8,stroke:#5f5e5a
    style SvcV1 fill:#e1f5ee,stroke:#0f6e56
    style SvcV2 fill:#e1f5ee,stroke:#0f6e56
    style NOTE fill:#faece7,stroke:#993c1d
```

## Bild-Quelltext (Istio)

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

## Bild-Quelltext (Kubernetes Gateway API)

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
