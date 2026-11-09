# Uebung: Bookinfo mit HTTPS und Let's Encrypt

## Ziel

Bookinfo ist danach unter `https://bookinfo.<dein-user>.do.t3isp.de/productpage` mit einem echten Zertifikat erreichbar,
`http://` leitet auf `https://` um. cert-manager holt das Zertifikat bei Let's Encrypt (HTTP-01) und erneuert es automatisch.

## Voraussetzung

  * Uebung [Bookinfo demo in ambient ausrollen](../installation/ambient/04-install-demo-app.md) ist durchgefuehrt
    (Namespace `bookinfo`, Gateway `bookinfo-gateway`, HTTPRoute `bookinfo`).
  * Der DNS-Eintrag `bookinfo.<dein-user>.do.t3isp.de` existiert schon (siehe Merkkasten).

> **Merkkasten: DNS ist schon eingerichtet**
>
> Der Trainer hat fuer jeden Cluster vorab einen DNS-Eintrag angelegt, z. B. `bookinfo.tln5.do.t3isp.de`.
> Er zeigt auf die oeffentliche IP deines ersten Workers. Du musst **nichts** im DNS eintragen.
> Die IP holst du dir in Schritt 1 per `dig` und gibst sie dem Gateway mit.

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir letsencrypt
cd letsencrypt
```

```
export HOST=bookinfo.$(whoami).do.t3isp.de
export IP=$(dig +short $HOST)
echo $HOST $IP
```

Beide Werte muessen gefuellt sein. Ist `IP` leer: Trainer fragen.

## Schritt 2: cert-manager installieren

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```
helm install cert-manager jetstack/cert-manager \
  --version v1.21.3 \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true \
  --set config.apiVersion=controller.config.cert-manager.io/v1alpha1 \
  --set config.kind=ControllerConfiguration \
  --set config.enableGatewayAPI=true \
  --wait
```

```
kubectl -n cert-manager get pods
```

Erwartet: 3 Pods `Running`.

## Schritt 3: Issuer anlegen

```
# vi 01-issuer.yml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - kind: Gateway
            name: bookinfo-gateway
```

```
kubectl -n bookinfo apply -f 01-issuer.yml
kubectl -n bookinfo get issuer
```

Erwartet: `READY True`.

## Schritt 4: Gateway mit HTTPS

```
# vi 02-gateway.yml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  annotations:
    cert-manager.io/issuer: letsencrypt
spec:
  gatewayClassName: istio
  addresses:
  - type: IPAddress
    value: ${IP}
  listeners:
  - name: http
    hostname: ${HOST}
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
  - name: https
    hostname: ${HOST}
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: bookinfo-tls
    allowedRoutes:
      namespaces:
        from: Same
```

```
envsubst < 02-gateway.yml | kubectl -n bookinfo apply -f -
```

```
kubectl -n bookinfo get gateway bookinfo-gateway
```

Erwartet: `PROGRAMMED True`, `ADDRESS` = deine IP.

## Schritt 5: Zertifikat abwarten

```
kubectl -n bookinfo get certificate
```

Wiederholen, bis `READY True` (ca. 1 Minute).

## Schritt 6: HTTP auf HTTPS umleiten

```
# vi 03-httproute-redirect.yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-to-https
spec:
  parentRefs:
  - name: bookinfo-gateway
    sectionName: http
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

```
# vi 04-httproute-bookinfo.yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo
spec:
  parentRefs:
  - name: bookinfo-gateway
    sectionName: https
  rules:
  - matches:
    - path:
        type: Exact
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: Exact
        value: /login
    - path:
        type: Exact
        value: /logout
    - path:
        type: PathPrefix
        value: /api/v1/products
    backendRefs:
    - name: productpage
      port: 9080
```

```
kubectl -n bookinfo apply -f 03-httproute-redirect.yml -f 04-httproute-bookinfo.yml
```

## Schritt 7: Testen

```
curl -s https://$HOST/productpage | grep -o "<title>.*</title>"
```

Erwartet: `<title>Simple Bookstore App</title>`

```
curl -sI http://$HOST/productpage
```

Erwartet: `HTTP/1.1 301 Moved Permanently` und `location: https://...`

```
echo https://$HOST/productpage
```

Die URL im Browser oeffnen: Schloss in der Adressleiste, Aussteller `Let's Encrypt`.

## Fehlersuche

| Beobachtung | Pruefen |
|-------------|---------|
| `Certificate` bleibt `READY False` | `kubectl -n bookinfo describe challenge` |
| Gateway nicht `PROGRAMMED True` | `IP` gesetzt? `echo $IP` |
| Nichts passiert nach dem Gateway | cert-manager mit `config.enableGatewayAPI=true` installiert? |
| Let's Encrypt meldet `too many ...` | Limit der CA erreicht (5 Fehlversuche pro Stunde je Hostname), spaeter erneut |

## Aufraeumen

```
kubectl -n bookinfo delete httproute http-to-https
kubectl -n bookinfo delete gateway bookinfo-gateway
kubectl -n bookinfo delete httproute bookinfo
kubectl -n bookinfo delete issuer letsencrypt
kubectl -n bookinfo delete secret bookinfo-tls letsencrypt-account-key
```

```
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
kubectl get crd -o name | grep cert-manager.io | xargs kubectl delete
```

Bookinfo wieder ohne HTTPS:

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

## Ref

  * https://cert-manager.io/docs/usage/gateway/
