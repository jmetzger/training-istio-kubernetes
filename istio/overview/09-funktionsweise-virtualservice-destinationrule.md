# Istio: VirtualService & DestinationRule

## Überblick

| Ressource | Aufgabe |
|-----------|---------|
| **VirtualService** | *Wohin* wird Traffic geroutet? (Routing-Regeln, Matching, Rewrites) |
| **DestinationRule** | *Wie* genau geht der Traffic? (Subsets, Load Balancing, TLS, Connection Pools) |

Beide arbeiten zusammen: Der VirtualService wählt das Ziel, die DestinationRule definiert die Eigenschaften des Ziels.

---

## VirtualService

Ein VirtualService fängt Traffic ab und leitet ihn anhand von Regeln weiter.

### Routing-Möglichkeiten

| Feature | Beschreibung |
|---------|-------------|
| Traffic Splitting (Canary) | Über `weight` auf mehrere Subsets verteilen |
| Header-basiertes Routing | `match.headers` mit `exact`, `prefix`, `regex` |
| URI-basiertes Routing | `match.uri` mit `exact`, `prefix`, `regex` |
| Fault Injection | `fault.delay` und `fault.abort` mit Prozentwerten |
| Timeouts & Retries | `timeout`, `retries.attempts`, `retries.perTryTimeout`, `retries.retryOn` |

---

## DestinationRule

Eine DestinationRule definiert Policies **nach** dem Routing (Subsets, Load Balancing, Circuit Breaking, TLS).

### Traffic Policies

| Feature | Feld | Optionen |
|---------|------|----------|
| Load Balancing | `trafficPolicy.loadBalancer.simple` | `ROUND_ROBIN`, `RANDOM`, `LEAST_CONN`, `PASSTHROUGH` |
| Consistent Hashing | `trafficPolicy.loadBalancer.consistentHash` | `httpHeaderName`, `httpCookie`, `useSourceIp` |
| Circuit Breaking | `trafficPolicy.connectionPool` | `tcp.maxConnections`, `http.http1MaxPendingRequests`, `http.http2MaxRequests` |
| Outlier Detection | `trafficPolicy.outlierDetection` | `consecutive5xxErrors`, `interval`, `baseEjectionTime`, `maxEjectionPercent` |
| mTLS | `trafficPolicy.tls.mode` | `DISABLE`, `SIMPLE`, `MUTUAL`, `ISTIO_MUTUAL` |

Subset-spezifische `trafficPolicy` überschreibt die globale Policy nur für das jeweilige Subset.

---

## Zusammenspiel

1. **DestinationRule** erstellen → Subsets definieren (z.B. `v1`, `v2`, `v3`) + globale Policies (TLS, LB)
2. **VirtualService** erstellen → Routing-Regeln mit Verweis auf die Subsets

---

## Cheat Sheet

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
