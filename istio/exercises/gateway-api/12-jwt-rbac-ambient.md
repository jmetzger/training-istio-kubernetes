# JWT-Token mit RBAC verwenden (Ambient)

> Das ist die **Ambient**-Fassung der Übung. Die Fassung für Sidecars steht in
> [12-jwt-rbac.md](../12-jwt-rbac.md).

## Was ist im Ambient-Mode anders?

Die JWT-Prüfung (`RequestAuthentication`) und Regeln auf JWT-Claims (`requestPrincipals`, `request.auth.claims`) sind **Layer-7-Funktionen** (HTTP).
Der **ztunnel** kann im Ambient-Mode nur Layer 4. Layer 7 erledigt der **Waypoint**.

| | Sidecar | Ambient |
|---|---|---|
| Wer prüft das JWT? | Envoy-Sidecar im Pod von httpbin | **Waypoint** |
| Policy zuordnen mit | `selector` (Pod-Label) | **`targetRefs`** (Service oder Waypoint-Gateway) |
| Workloads | `istioctl kube-inject` | kein Inject, der Namespace hat `istio.io/dataplane-mode=ambient` |

> [!WARNING]
> Nimmst du wie in der Sidecar-Übung `selector`, wirkt die Policy **nicht wie gewünscht**:
> Eine `RequestAuthentication` mit `selector` prüft kein JWT (alles liefert `200`),
> und eine `AuthorizationPolicy` mit `selector` und `requestPrincipals` wird vom ztunnel ausgewertet.
> Der ztunnel kennt keine HTTP-Attribute, lässt die Regel weg, und der Service ist für alle gesperrt (`503`).
> Das probierst du am Ende in "Step 13" selbst aus.

## Voraussetzungen

  * Übung [Bookinfo demo in ambient ausrollen](../../installation/ambient/04-install-demo-app.md) ist durchgeführt
    (Namespace `bookinfo` mit Waypoint, alle Pods `Running`).
  * Der Namespace ist im Ambient-Mode **und** nutzt den Waypoint:

```
kubectl get ns bookinfo --show-labels
```

Erwartet: `istio.io/dataplane-mode=ambient` und `istio.io/use-waypoint=waypoint`

  * Es gibt noch keine JWT-Policies im Namespace:

```
kubectl -n bookinfo get requestauthentication,authorizationpolicy
```

Erwartet: `No resources found`. Sonst vorher löschen.

## Grundlagen 

### Was ist JWKS ? 

**JWKS = JSON Web Key Set**

Ein JSON-Dokument, das die **öffentlichen Schlüssel** enthält, mit denen JWTs (JSON Web Tokens) verifiziert werden können.

**Typisches Format:**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "abc123",
      "use": "sig",
      "n": "0vx7agoebGc...",
      "e": "AQAB"
    }
  ]
}
```

**Wie es funktioniert:**
1. Ein Identity Provider (Keycloak, Auth0, Google etc.) signiert JWTs mit seinem **Private Key**
2. Der JWKS-Endpunkt (z.B. `https://idp.example.com/.well-known/jwks.json`) stellt die zugehörigen **Public Keys** bereit
3. Istio (oder ein anderer Verifier) holt sich die Keys von dort und prüft damit die JWT-Signatur

**In Istio konkret** — `RequestAuthentication`:
```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
spec:
  jwtRules:
  - issuer: "https://idp.example.com"
    jwksUri: "https://idp.example.com/.well-known/jwks.json"
```

Istio cached die Keys und rotiert automatisch mit, wenn der IDP neue Keys veröffentlicht (via `kid` — Key ID).

**Kurz:** JWKS ist der standardisierte Weg, wie ein JWT-Verifier an die Public Keys kommt, ohne sie manuell konfigurieren zu müssen.

### Begriffe 

**`kid`** (Key ID) — Eindeutige ID des Schlüssels. Damit weiß der Verifier, welcher Key aus dem Set zum JWT passt (das JWT hat `kid` im Header).

**`use`** (Public Key Use) — Wofür der Key gedacht ist:
- `"sig"` = Signaturprüfung (Standard bei JWT)
- `"enc"` = Verschlüsselung

**`n`** (Modulus) — Der RSA-Modulus, Base64url-kodiert. Das ist der mathematische Kern des öffentlichen RSA-Schlüssels.

**`e`** (Exponent) — Der RSA-Exponent, Base64url-kodiert. Fast immer `"AQAB"` (= 65537 dezimal), der Standard-Public-Exponent.

**Zusammenspiel:** `n` und `e` zusammen **sind** der öffentliche RSA-Schlüssel. Damit kann die JWT-Signatur mathematisch verifiziert werden, ohne den Private Key zu kennen.



## Step 0: Preparation 

```
cd
mkdir -p manifests/jwt
cd manifests/jwt 
```

## Step 1: Create http-bin and curl workloads 

```
kubectl -n bookinfo apply -f ~/istio/samples/httpbin/httpbin.yaml
kubectl -n bookinfo apply -f ~/istio/samples/curl/curl.yaml
kubectl -n bookinfo rollout status deploy/httpbin deploy/curl
```

## Step 2: Can we connect ? 

  * Erwartet: `200`

```
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl http://httpbin.bookinfo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
```

## Step 3: Create a RequestAuthentication 

```
nano 01-ra.yml 
```

```
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: "jwt-example"
  namespace: bookinfo
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: httpbin
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/jwks.json"
```

```
kubectl apply -f . 
```

>[!NOTE]
>`targetRefs` mit `kind: Service` bindet die Policy an den **Waypoint**, der für den Service `httpbin` zuständig ist.
>Mit `selector` (wie bei Sidecars) würde die Policy nicht greifen.

## Step 4: Check with an invalid jwt 

  * Invalid is restricted, so we do not get access (no 200) 
  * Erwartet: `401`

```
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer invalidToken" -w "%{http_code}\n"
```

## Step 5: But: without a jwt -> its work 

  * ... Because ! -> There is no AuthorizationPolicy
  * Erwartet: `200`

```
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
```

## Step 6: We create an AuthorizationPolicy 

>[!NOTE]
>requestPrincipal set to testing@secure.istio.io/testing@secure.istio.io. Istio constructs the requestPrincipal by combining the iss and sub of the JWT token with a / separator.
>
>iss = issuer
>sub = subject 

```
nano 02-ap.yml
```

```
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: bookinfo
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
```

```
kubectl apply -f 02-ap.yml
```

Prüfen, ob der Waypoint die Policy übernommen hat:

```
kubectl -n bookinfo get authorizationpolicy require-jwt -o jsonpath='{.status.conditions[*].message}{"\n"}'
```

Erwartet: `bound to bookinfo/waypoint`

## Step 7: Test access 

  * jwt consists of 3 parts
    * HEADER / PAYLOAD / SIGNATURE
    * Each part is base64 encoded
  * cut -d. -f2 -> gets the 2nd part -> the payload 
   
```
# This is the way we get the token
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode
```

```
echo $TOKEN
```

```
# Testing with allowed jwt  (erwartet: 200)
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
```

```
# Testing without a jwt  (erwartet: 403)
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
```

## Step 8: Update AuthorizationPolicy also needing a specific group 

```
nano 02-ap-group.yml
```

```
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: bookinfo
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group1"]
```

```
kubectl apply -f 02-ap-group.yml
```

## Step 9: get token included a claim for a group 

* Get the JWT that sets the groups claim to a list of strings: group1 and group2:

```
TOKEN_GROUP=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/groups-scope.jwt -s) && echo "$TOKEN_GROUP" | cut -d '.' -f2 - | base64 --decode
```

## Step 10: Test it with that token (so group1 must be included) 

  * Erwartet: `200`

```
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN_GROUP" -w "%{http_code}\n"
```

## Step 11: Test with a token without group included 

  * We use that TOKEN before, which had not group 
  * Erwartet: `403`

```
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
```

## Step 12: Overview of the results

| Aufruf | Nur RequestAuthentication | + AuthorizationPolicy | + Gruppe `group1` |
|---|---|---|---|
| ohne Token | `200` | `403` | `403` |
| ungültiges Token | `401` | `401` | `401` |
| gültiges Token (ohne Gruppe) | `200` | `200` | `403` |
| Token mit `group1` | `200` | `200` | `200` |

  * `401`: Das JWT ist ungültig, die Prüfung der `RequestAuthentication` im Waypoint lehnt ab.
  * `403`: Das JWT ist gültig oder fehlt, aber die `AuthorizationPolicy` erlaubt es nicht.

## Step 13: Stolperstein: selector statt targetRefs (optional)

Jetzt siehst du selbst, was bei Sidecars funktioniert und im Ambient-Mode nicht.
Erst die Policies aus den Steps davor löschen:

```
kubectl -n bookinfo delete authorizationpolicy/require-jwt requestauthentication/jwt-example
```

Die `AuthorizationPolicy` mit `selector` (wie in der Sidecar-Übung):

```
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt-selector
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
EOF
```

Der Status verrät das Problem:

```
kubectl -n bookinfo get authorizationpolicy require-jwt-selector -o jsonpath='{.status.conditions[*].message}{"\n"}'
```

Erwartet: `ztunnel does not support HTTP attributes (found: requestPrincipals). In ambient mode you must use a waypoint proxy to enforce HTTP rules. ...`

Auch das gültige Token kommt jetzt nicht mehr durch (erwartet: `503`):

```
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.29.1/security/tools/jwt/samples/demo.jwt -s)
kubectl exec "$(kubectl get pod -l app=curl -n bookinfo -o jsonpath={.items..metadata.name})" -c curl -n bookinfo -- curl "http://httpbin.bookinfo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
```

**Merke:** Im Ambient-Mode gehören Layer-7-Policies (JWT, HTTP-Methoden, Claims) per `targetRefs` an den Waypoint.

## Step 14: Cleanup 

```
kubectl -n bookinfo delete authorizationpolicy/require-jwt-selector
kubectl -n bookinfo delete -f ~/istio/samples/httpbin/httpbin.yaml -f ~/istio/samples/curl/curl.yaml
cd
```


## Reference: 

  * https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/
  * JWT Debugger: https://www.jwt.io/
