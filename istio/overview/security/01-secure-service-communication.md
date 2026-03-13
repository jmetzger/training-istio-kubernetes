# Sichere Service-Kommunikation mit Istio

## Überblick

Sichere Kommunikation im Istio Service Mesh basiert auf drei Säulen:

| Säule | Ressource | Funktion |
|-------|-----------|----------|
| Verschlüsselung & Identität | `PeerAuthentication` | mTLS zwischen Services |
| Zugriffskontrolle | `AuthorizationPolicy` | Wer darf mit wem kommunizieren |
| End-User-Authentifizierung | `RequestAuthentication` | JWT-Token-Validierung |

Erst das Zusammenspiel aller drei ergibt ein vollständiges Zero-Trust-Modell.

---

## 1. mTLS mit PeerAuthentication

### Was passiert unter der Haube?

- Istio stellt jedem Workload automatisch ein **X.509-Zertifikat** aus (SPIFFE-ID)
- Format: `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`
- Zertifikate werden über die **Istio CA (istiod)** ausgestellt und regelmäßig rotiert
- Der Traffic zwischen Sidecars (Classic Mode) bzw. ztunnel (Ambient Mode) wird transparent verschlüsselt

### Modi

| Modus | Verhalten |
|-------|-----------|
| `PERMISSIVE` | Akzeptiert mTLS **und** Plaintext (Default, gut für Migration) |
| `STRICT` | **Nur** mTLS erlaubt — Plaintext wird abgelehnt |
| `DISABLE` | mTLS deaktiviert |
| `UNSET` | Erbt Einstellung vom Parent (Mesh → Namespace → Workload) |

### Beispiel: Mesh-weit STRICT

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

### Beispiel: Namespace-Level

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Beispiel: Port-Level Exception

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: my-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-service
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: PERMISSIVE  # Health-Check-Port ohne mTLS
```

### Vererbungshierarchie

```
Mesh-Policy (istio-system)
  └── Namespace-Policy
        └── Workload-Policy (mit selector)
              └── Port-Level Override
```

Die spezifischste Policy gewinnt.

---

## 2. Zugriffskontrolle mit AuthorizationPolicy

### Warum reicht mTLS nicht?

mTLS stellt sicher, **wer** ein Service ist (Identität) und verschlüsselt den Traffic. Aber ohne AuthorizationPolicy kann **jeder** authentifizierte Service **jeden** anderen erreichen. Das ist kein Zero Trust.

### Aktionstypen

| Action | Verhalten |
|--------|-----------|
| `ALLOW` | Erlaubt Zugriff wenn Regeln matchen |
| `DENY` | Verweigert Zugriff wenn Regeln matchen (hat Vorrang vor ALLOW) |
| `CUSTOM` | Delegiert Entscheidung an externen Authorizer |

### Evaluierungsreihenfolge

```
1. CUSTOM → 2. DENY → 3. ALLOW → 4. Implizites Verhalten
```

Wenn **keine** AuthorizationPolicy existiert → alles erlaubt.  
Wenn **nur** ALLOW-Policies existieren → alles andere wird implizit verweigert.

### Beispiel: Nur frontend darf auf backend zugreifen

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

### Beispiel: Deny-Rule für sensible Endpoints

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: deny-admin
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: DENY
  rules:
  - to:
    - operation:
        paths: ["/admin*"]
    from:
    - source:
        notPrincipals: ["cluster.local/ns/production/sa/admin-service"]
```

### Wichtige Source-Felder (L4)

| Feld | Beschreibung |
|------|-------------|
| `principals` | SPIFFE-ID des aufrufenden Service |
| `namespaces` | Source-Namespace |
| `ipBlocks` | Source-IP-Bereiche |

### Wichtige Operation-Felder (L7)

| Feld | Beschreibung |
|------|-------------|
| `methods` | HTTP-Methoden (GET, POST, ...) |
| `paths` | URL-Pfade |
| `hosts` | Ziel-Hostname |
| `ports` | Ziel-Ports |

> **Hinweis:** L7-Felder (methods, paths, hosts) erfordern im Ambient Mode einen **Waypoint Proxy**. Der ztunnel arbeitet nur auf L4.

---

## 3. End-User-Authentifizierung mit RequestAuthentication

### Wofür?

PeerAuthentication sichert **Service-zu-Service**. RequestAuthentication validiert **End-User-Tokens** (JWTs), die von einem externen Identity Provider stammen (z.B. Keycloak, Auth0, Dex).

### Beispiel: JWT-Validierung

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  jwtRules:
  - issuer: "https://keycloak.example.com/realms/myrealm"
    jwksUri: "https://keycloak.example.com/realms/myrealm/protocol/openid-connect/certs"
    forwardOriginalToken: true
```

### Verhalten

- Request **ohne** Token → wird **durchgelassen** (RequestAuthentication prüft nur vorhandene Tokens!)
- Request **mit gültigem** Token → wird durchgelassen, Claims werden extrahiert
- Request **mit ungültigem** Token → wird abgelehnt (401)

### Kombination mit AuthorizationPolicy

Um Requests **ohne** Token abzulehnen, braucht man eine ergänzende AuthorizationPolicy:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["https://keycloak.example.com/realms/myrealm/*"]
    when:
    - key: request.auth.claims[groups]
      values: ["admin", "editor"]
```

> **Wichtig:** `RequestAuthentication` allein erzwingt **keine** Authentifizierung. Erst zusammen mit einer `AuthorizationPolicy` auf `requestPrincipals` wird ein Token tatsächlich **erforderlich**.

---

## Zusammenspiel der drei Säulen

```
                     Eingehender Request
                            │
                ┌───────────┴───────────┐
                │  PeerAuthentication   │
                │  (mTLS-Identität)     │
                └───────────┬───────────┘
                            │
                ┌───────────┴───────────┐
                │ RequestAuthentication │
                │ (JWT-Validierung)     │
                └───────────┬───────────┘
                            │
                ┌───────────┴───────────┐
                │  AuthorizationPolicy  │
                │  (Zugriffskontrolle)  │
                └───────────┬───────────┘
                            │
                      ┌─────┴─────┐
                      │  Service  │
                      └───────────┘
```

### Typische Konfigurationsreihenfolge

1. **PeerAuthentication** auf `STRICT` setzen (Mesh-weit oder Namespace)
2. **AuthorizationPolicy** erstellen (wer darf wohin)
3. **RequestAuthentication** hinzufügen (wenn End-User-Tokens validiert werden sollen)
4. **AuthorizationPolicy** mit `requestPrincipals` ergänzen (Token erzwingen + Claims prüfen)

---

## Debugging & Verifizierung

### mTLS-Status prüfen

```bash
# Prüfen ob mTLS aktiv ist
istioctl x describe pod <pod-name> -n <namespace>

# PeerAuthentication-Policies anzeigen
kubectl get peerauthentication --all-namespaces

# TLS-Verbindung verifizieren
istioctl proxy-config secret <pod-name> -n <namespace>
```

### AuthorizationPolicy testen

```bash
# RBAC-Debug-Logs aktivieren
istioctl proxy-config log <pod-name> --level rbac:debug

# Logs prüfen
kubectl logs <pod-name> -c istio-proxy | grep rbac

# Typische Log-Einträge:
# "enforced denied" → Policy hat geblockt
# "enforced allowed" → Policy hat erlaubt
```

### JWT-Probleme debuggen

```bash
# Token manuell prüfen
kubectl exec <pod-name> -c istio-proxy -- \
  curl -s http://localhost:15000/config_dump | \
  jq '.configs[] | select(.["@type"] | contains("listener"))'

# JWT-Rejection im Access-Log
# Response-Flag "UAEX" = Unauthorized External (JWT ungültig)
```

---

## Classic Mode vs. Ambient Mode

| Aspekt | Classic (Sidecar) | Ambient (ztunnel + Waypoint) |
|--------|-------------------|------------------------------|
| mTLS | Sidecar-zu-Sidecar | ztunnel-zu-ztunnel |
| L4-Policy | Im Sidecar | Im ztunnel |
| L7-Policy | Im Sidecar | **Nur mit Waypoint Proxy** |
| RequestAuthentication | Im Sidecar | **Nur mit Waypoint Proxy** |
| Zertifikate | Pro Pod | Pro Node (ztunnel) / Pro SA (Waypoint) |

> **Praxis-Tipp:** Im Ambient Mode zunächst nur L4-Policies (principals, namespaces) nutzen. Für L7 (paths, methods, JWT) einen Waypoint deployen.
