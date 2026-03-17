# Gateway API als Nachfolger von IngressGateway (Istio), VirtualService und DestinationRule 

## Wichtig Hinweis:

  * In Istio werden immer mehr Sachen von der Gateway API übernommen
  * Aber noch nicht alles ist migriert, deswegen brauche ich telweise immer noch die Objekte (VirtualService und DestinationRule für bestimmte Features

## Was Gateway API ersetzt 

## Kurze Antwort: Für Routing nein, für einige Istio-spezifische Features ja.

### Was Gateway API vollständig ersetzt (Stand: 03/2026)

| Feature | VirtualService | Gateway API Equivalent |
|---|---|---|
| Request Routing (Header, Path, Method) | ✅ | HTTPRoute `matches` |
| Traffic Splitting / Canary | ✅ | HTTPRoute `backendRefs` mit `weight` |
| URL Rewrite | ✅ | HTTPRoute `URLRewrite` Filter |
| Request Header Modification | ✅ | HTTPRoute `RequestHeaderModifier` |
| Redirects | ✅ | HTTPRoute `RequestRedirect` |
| Request Mirroring | ✅ | HTTPRoute `RequestMirror` |
| gRPC Routing | ✅ | GRPCRoute |

### Was du weiterhin als Istio-API brauchst

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

### Praxis-Empfehlung

```
Neues Projekt:
  Routing            → Gateway API (HTTPRoute)
  Fault Injection    → VirtualService (solange nötig)
  Backend-Policies   → DestinationRule
  
Bestehendes Projekt:
  Migration schrittweise, beides kann parallel existieren
```

**Achtung:** Nicht beide gleichzeitig (httpRoute und VirtualService) auf denselben Service anwenden — das führt zu undefiniertem Verhalten. Entweder HTTPRoute **oder** VirtualService pro Service, nicht mischen.


