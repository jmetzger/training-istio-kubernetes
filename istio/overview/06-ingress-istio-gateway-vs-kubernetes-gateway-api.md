# Ingress Istio-Gateway vs. Kubernetes Gateway API 

## Istio Ingress-Gateway-Flow (bookinfo/productpage) 

```mermaid
graph LR
    Client[Client/Browser]
    IG[Istio IngressGateway<br/>Envoy Pod]
    GW[Gateway<br/>networking.istio.io]
    VS[VirtualService<br/>routing rules]
    ROUTES["route: host=reviews, subset=v1 (80%)<br/>route: host=reviews, subset=v2 (20%)"]
    DR[DestinationRule reviews<br/>host: reviews — defines subsets via labels]
    V1[reviews v1<br/>version: v1]
    V2[reviews v2<br/>version: v2]
    V3[reviews v3<br/>version: v3]

    Client -->|HTTP/HTTPS| IG
    IG -.->|references| GW
    GW -.->|bound to| VS
    VS --> ROUTES
    ROUTES -.->|references subsets| DR
    DR -->|"80%"| V1
    DR -->|"20%"| V2
    DR -->|"0%"| V3

    style IG fill:#e1f5ff,stroke:#185fa5
    style GW fill:#fff3cd,stroke:#854f0b
    style VS fill:#fff3cd,stroke:#854f0b
    style ROUTES fill:#f1efe8,stroke:#5f5e5a
    style DR fill:#faece7,stroke:#993c1d
    style V1 fill:#e1f5ee,stroke:#0f6e56
    style V2 fill:#e1f5ee,stroke:#0f6e56
    style V3 fill:#e1f5ee,stroke:#0f6e56
```

## Istio Gateway

<img width="1267" height="184" alt="image" src="https://github.com/user-attachments/assets/41d95be5-8713-4533-9821-ea8e2ed5a0f5" />

## Gateway API 

<img width="1024" height="703" alt="image" src="https://github.com/user-attachments/assets/a8be8bb3-a761-4fc2-b482-9a6798b49eda" />

   * Achtung:  VirtualService und Destination werden teilweise immer noch benötigt, auch wenn man mit der Gateway API arbeitet,
     um bestimmte Feature abzubilden  

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

```mermaid
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
