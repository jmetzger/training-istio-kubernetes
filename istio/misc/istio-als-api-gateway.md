# Istio als API-Gateway 

## Was sind Dinge, die istio nicht unterstützt

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

## Was sind Features, die istio unterstützt 

**Was oft verwechselt wird — das kann Istio durchaus:**
- Rate Limiting (via WASM oder EnvoyFilter)
- JWT-Validierung / OIDC
- Header-basiertes Routing
- mTLS / AuthorizationPolicy

**Kernaussage:** Istio fehlt vor allem alles rund um **Body-Manipulation, Schema-Validierung und API-als-Produkt-Features**. Auf der Traffic- und Security-Ebene ist es sehr vollständig.
