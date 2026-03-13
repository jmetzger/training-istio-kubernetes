# Header-Größenlimits: Istio/Envoy vs. nginx (Keycloak-Kontext)

## Problem

Keycloak erzeugt große HTTP-Header (JWTs, Set-Cookie mit Access/ID/Refresh Token). Sowohl nginx als auch Istio/Envoy haben Default-Limits, die dabei überschritten werden können.

---

## Envoy/Istio Defaults

| Parameter                 | Default  | Maximum  |
|---------------------------|----------|----------|
| `max_request_headers_kb`  | 60 KiB*  | **96 KiB** (hartes Limit) |
| `max_headers_count`       | 100      | frei konfigurierbar |

> **Achtung:** In älteren Istio-Versionen (< 1.8) lag der tatsächliche Default im Code bei ~29 KiB, nicht bei den dokumentierten 60 KiB.

**Fehlermeldung bei Überschreitung:** `431 Request Header Fields Too Large`

## nginx Defaults

| Parameter                    | Default     |
|------------------------------|-------------|
| `large_client_header_buffers`| 4 × 8 KB   |
| `proxy_buffer_size`          | 4k / 8k     |
| `proxy_buffers`              | 8 × 4k/8k  |

**Fehlermeldungen bei Überschreitung:**

- **400 Bad Request** — Request-Header des Clients zu groß (`large_client_header_buffers`)
- **502 Bad Gateway** — Response-Header vom Upstream (Keycloak) zu groß (`proxy_buffer_size`) — der Klassiker mit Keycloak

---

## Fix: Istio/Envoy (EnvoyFilter)

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

### Bekannte Probleme

- Das **Verringern** des Limits funktioniert zuverlässig, das **Erhöhen** über den Default hinaus greift manchmal nicht (gemeldetes Istio-Issue)
- Werte > 96 werden ignoriert — **128 ist ungültig!**
- Der `@type` muss zur Istio-Version passen:
  - Istio ≤ 1.6: `...http_connection_manager.v2.HttpConnectionManager`
  - Istio ≥ 1.8: `...http_connection_manager.v3.HttpConnectionManager`

---

## Fix: nginx

```nginx
# Response-Header vom Upstream (Keycloak) — behebt 502
proxy_buffer_size       128k;
proxy_buffers           4 256k;
proxy_busy_buffers_size 256k;

# Request-Header vom Client — behebt 400
large_client_header_buffers 4 32k;
```

---

## Checkliste: Weitere Header-Limits prüfen

Nicht nur der Proxy, auch die **Anwendung selbst** kann Header ablehnen:

| Framework / Server | Default Max Header |
|--------------------|--------------------|
| Spring Boot        | 8 KiB              |
| Gunicorn           | 8 KiB              |
| Tomcat             | 8 KiB              |
| Node.js (http)     | 16 KiB             |

Falls nach dem EnvoyFilter-Fix weiterhin Fehler auftreten, liegt es oft am App-Container, nicht an Envoy.
