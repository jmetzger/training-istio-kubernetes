# Uebung: Bookinfo mit HTTPS und Let's Encrypt (cert-manager, Gateway API, Ambient)

## Ziel

Die Bookinfo-App aus der Ambient-Uebung ist bisher nur per HTTP erreichbar. Danach erreichst du sie unter
`https://bookinfo-<dein-user>.do.t3isp.de` mit einem **echten, vom Browser anerkannten Zertifikat**, und `http://` leitet automatisch auf `https://` um.

## Voraussetzung

  * Die Uebung [Bookinfo demo in ambient ausrollen](../installation/ambient/04-install-demo-app.md) ist komplett durchgefuehrt:
    Namespace `bookinfo` mit Waypoint, Bookinfo-App, `bookinfo-gateway` und HTTPRoute `bookinfo`.
  * Die Trainer haben fuer jeden Cluster einen DNS-Eintrag angelegt: `bookinfo-<dein-user>.do.t3isp.de` zeigt auf die erste
    IP aus dem MetalLB-Pool deines Clusters (z. B. `bookinfo-tln5.do.t3isp.de`). Du musst nichts eintragen.

## Hintergrund

| Baustein | Aufgabe |
|----------|---------|
| Let's Encrypt | Kostenlose Zertifizierungsstelle (CA), Zertifikate 90 Tage gueltig |
| cert-manager | Holt und erneuert die Zertifikate automatisch im Cluster |
| `Issuer` | Sagt cert-manager, bei welcher CA und mit welcher Challenge er Zertifikate holt (gilt nur im Namespace) |
| HTTP-01-Challenge | Let's Encrypt ruft `http://<hostname>/.well-known/acme-challenge/<token>` auf. Klappt das, gehoert dir der Hostname |
| Gateway-Annotation | `cert-manager.io/issuer` am Gateway: cert-manager legt das `Certificate` und das TLS-Secret selbst an |

Ablauf:

  1. Du traegst am Gateway einen HTTPS-Listener mit Hostname ein und setzt die Annotation.
  2. cert-manager erzeugt ein `Certificate`, daraus eine `Order` und eine `Challenge`.
  3. Fuer die Challenge legt cert-manager **kurzzeitig eine eigene HTTPRoute** am Gateway an, damit Let's Encrypt das Token ueber Port 80 findet.
  4. Let's Encrypt prueft, cert-manager speichert das Zertifikat im Secret `bookinfo-tls`, das Gateway nutzt es.

Ambient: Das TLS endet **am Gateway** (Verkehr von aussen). Ab dort uebernimmt Ambient die Absicherung im Mesh
(ztunnel, mTLS ueber HBONE). Am Mesh und am Waypoint aendert diese Uebung nichts.

**Staging zuerst:** Let's Encrypt hat Limits (u. a. 5 gleiche Zertifikate pro Woche und 5 fehlgeschlagene Pruefungen pro Stunde je Hostname).
Deshalb testen wir erst gegen die Staging-CA (gleiche Technik, aber unbegrenzt und mit unbekanntem Aussteller) und wechseln dann auf die echte CA.

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir letsencrypt
cd letsencrypt
```

```
# Hostname und IP fuer deinen Cluster (Variablen gelten nur in diesem Terminal)
export HOST=bookinfo-$(whoami).do.t3isp.de
export IP=$(kubectl -n metallb-system get ipaddresspool first-pool -o jsonpath='{.spec.addresses[0]}' | sed 's#/32##')
echo $HOST $IP
```

```
# Der DNS-Eintrag muss auf genau diese IP zeigen
dig +short $HOST
```

Beide Ausgaben muessen dieselbe IP zeigen. Wenn `dig` nichts liefert: Trainer fragen (DNS-Eintrag fehlt).

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

`config.enableGatewayAPI=true` schaltet die Unterstuetzung fuer Gateway-Ressourcen ein (ohne diese Option ignoriert cert-manager die Annotation am Gateway).

```
kubectl -n cert-manager get pods
```

Erwartet: 3 Pods `Running` (`cert-manager`, `cainjector`, `webhook`).

```
kubectl -n cert-manager logs deploy/cert-manager | grep -i "gateway api"
```

Erwartet: eine Zeile `enabling the sig-network Gateway API certificate-shim and HTTP-01 solver`.

## Schritt 3: Issuer fuer die Staging-CA

```
# vi 01-issuer-staging.yml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - kind: Gateway
            name: bookinfo-gateway
```

```
kubectl -n bookinfo apply -f 01-issuer-staging.yml
kubectl -n bookinfo get issuer
```

Erwartet: `letsencrypt-staging` mit `READY True` (nach wenigen Sekunden, cert-manager hat sich dann bei der CA registriert).

Der Issuer liegt im Namespace `bookinfo`. Er ist damit Sache des Namespace-Teams und braucht keine Rechte auf Cluster-Ebene
(das waere ein `ClusterIssuer`).

## Schritt 4: Gateway um HTTPS erweitern

```
# vi 02-gateway.yml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  annotations:
    cert-manager.io/issuer: letsencrypt-staging
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

Was ist neu gegenueber der Gateway-Datei aus der Bookinfo-Uebung:

  * `addresses`: Das Gateway bekommt genau die IP, auf die der DNS-Eintrag zeigt (sonst koennte MetalLB die andere Worker-IP vergeben).
  * `hostname` an beiden Listenern: Let's Encrypt stellt nur fuer diesen Namen aus.
  * Listener `https` auf Port 443, `tls.mode: Terminate`: Das Gateway beendet TLS, das Zertifikat liegt im Secret `bookinfo-tls`
    (das Secret gibt es noch nicht, cert-manager legt es an).
  * Annotation `cert-manager.io/issuer`: Hier steckt der ganze Trick, cert-manager erzeugt daraus das `Certificate`.

`${IP}` und `${HOST}` setzt `envsubst` ein, deshalb nicht direkt mit `kubectl apply -f .`:

```
envsubst < 02-gateway.yml | kubectl -n bookinfo apply -f -
```

```
kubectl -n bookinfo get gateway bookinfo-gateway
kubectl -n bookinfo get svc bookinfo-gateway-istio
```

Erwartet: `PROGRAMMED True`, `EXTERNAL-IP` = deine IP, Ports `80` und `443` am selben Service (kein zweiter LoadBalancer noetig).

## Schritt 5: Zertifikat entsteht

```
kubectl -n bookinfo get certificate,order,challenge
```

Wiederholen, bis das `Certificate` `READY True` hat (rund 30 bis 60 Sekunden). Zwischendurch sieht man `Order` und `Challenge` im Zustand `pending`.

```
# Die Challenge-Route von cert-manager (nur waehrend der Pruefung vorhanden)
kubectl -n bookinfo get httproute
```

```
kubectl -n bookinfo describe certificate bookinfo-tls | tail -10
```

Am Ende der Events: `The certificate has been successfully issued`.

## Schritt 6: Testen (Staging)

```
# Ohne Option: Fehler, weil die Staging-CA kein Browser kennt
curl https://$HOST/productpage
```

**Erwarteter Fehler:**
```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

```
# -k: Pruefung des Zertifikats ueberspringen
curl -k -s https://$HOST/productpage | grep -o "<title>.*</title>"
```

Erwartet: `<title>Simple Bookstore App</title>`

```
# Wer hat das Zertifikat ausgestellt?
echo | openssl s_client -connect $HOST:443 -servername $HOST 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

Erwartet: `issuer=... CN = (STAGING) Ersatz Emmer ...` (Der Name der Staging-Zwischenzertifikate wechselt, wichtig ist `(STAGING)`), `subject=CN = bookinfo-<dein-user>.do.t3isp.de`.

## Schritt 7: Auf die echte Let's-Encrypt-CA wechseln

```
# vi 03-issuer-prod.yml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - kind: Gateway
            name: bookinfo-gateway
```

```
# In der Gateway-Datei die Annotation auf prod aendern
sed 's#cert-manager.io/issuer: letsencrypt-staging#cert-manager.io/issuer: letsencrypt-prod#' 02-gateway.yml > 04-gateway-prod.yml
diff 02-gateway.yml 04-gateway-prod.yml
```

```
kubectl -n bookinfo apply -f 03-issuer-prod.yml
envsubst < 04-gateway-prod.yml | kubectl -n bookinfo apply -f -
```

cert-manager bemerkt den neuen Aussteller und holt das Zertifikat neu (Events: `Issuing certificate as Secret was previously issued by "Issuer.cert-manager.io/letsencrypt-staging"`).

```
kubectl -n bookinfo get certificate
```

Wiederholen, bis `READY True` (rund eine Minute; zwischendurch `False` ist normal).

```
# Jetzt ohne -k
curl -s https://$HOST/productpage | grep -o "<title>.*</title>"
```

```
echo | openssl s_client -connect $HOST:443 -servername $HOST 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

Erwartet: `issuer=C = US, O = Let's Encrypt, CN = ...` (ohne `STAGING`), Gueltigkeit rund 90 Tage.

## Schritt 8: HTTP auf HTTPS umleiten

Die HTTPRoute `bookinfo` haengt noch an beiden Listenern. Wir binden sie an `https` und legen eine Umleitung fuer `http` an:

```
# vi 05-httproute-redirect.yml
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
# vi 06-httproute-bookinfo.yml
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
kubectl -n bookinfo apply -f 05-httproute-redirect.yml -f 06-httproute-bookinfo.yml
kubectl -n bookinfo get httproute
```

```
# Weiterleitung ansehen
curl -sI http://$HOST/productpage
```

Erwartet: `HTTP/1.1 301 Moved Permanently` und `location: https://bookinfo-<dein-user>.do.t3isp.de/productpage`.

```
# Weiterleitung folgen (-L)
curl -sL http://$HOST/productpage | grep -o "<title>.*</title>"
```

Die HTTP-01-Challenge von cert-manager (Pfad `/.well-known/acme-challenge/...`) ist spezifischer als die Weiterleitung auf `/`
und gewinnt deshalb. Auch die automatische Erneuerung funktioniert damit weiter.

## Schritt 9: Im Browser ansehen

`http://<dein-hostname>/productpage` im Browser oeffnen: Du landest auf `https://`, in der Adressleiste ist das Schloss, und ein Klick darauf zeigt
`Let's Encrypt` als Aussteller.

Hostname ausgeben lassen:

```
echo https://$HOST/productpage
```

## Schritt 10: Erneuerung anschauen

```
kubectl -n bookinfo get certificate bookinfo-tls -o jsonpath='{.status.notBefore}{"\n"}{.status.notAfter}{"\n"}{.status.renewalTime}{"\n"}'
```

`renewalTime` liegt rund 30 Tage vor Ablauf: Ab dann erneuert cert-manager das Zertifikat ohne dein Zutun.

## Fehlersuche

| Beobachtung | Ursache / Pruefen |
|-------------|-------------------|
| `Certificate` bleibt `READY False` | `kubectl -n bookinfo describe challenge` lesen, dort steht die Fehlermeldung der CA |
| Challenge `pending`, Meldung `connection refused` oder `timeout` | Gateway nicht `PROGRAMMED True` oder IP stimmt nicht mit DNS ueberein (`dig +short $HOST` gegen `EXTERNAL-IP`) |
| Nichts passiert nach dem Anwenden des Gateways | cert-manager ohne `config.enableGatewayAPI=true` installiert, oder Annotation falsch geschrieben |
| `EXTERNAL-IP` bleibt `<pending>` | Pro Cluster hoechstens zwei LoadBalancer-Services (zwei Worker-IPs), oder die IP aus `addresses` gehoert nicht zum Pool |
| `NXDOMAIN` bei `dig` | DNS-Eintrag fehlt, Trainer fragen |
| Let's Encrypt meldet `too many certificates` | Rate-Limit der echten CA erreicht: bis zum Ablauf der Sperre mit Staging weiterarbeiten |

## Aufraeumen

```
kubectl -n bookinfo delete httproute http-to-https
kubectl -n bookinfo delete gateway bookinfo-gateway
kubectl -n bookinfo delete httproute bookinfo
kubectl -n bookinfo delete issuer letsencrypt-staging letsencrypt-prod
kubectl -n bookinfo delete certificate bookinfo-tls
kubectl -n bookinfo delete secret bookinfo-tls letsencrypt-staging-account-key letsencrypt-prod-account-key
```

```
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
kubectl get crd -o name | grep cert-manager.io | xargs kubectl delete
```

Wer weiter mit Bookinfo arbeiten will, legt Gateway und Route aus der Bookinfo-Uebung wieder an:

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

## Zusammenfassung

| Schritt | Ergebnis |
|---------|----------|
| Staging-Issuer, Gateway mit Annotation | Zertifikat von `(STAGING)`, Browser warnt, `curl -k` geht |
| Wechsel auf `letsencrypt-prod` | Echtes Zertifikat, `curl` ohne `-k` geht |
| HTTPRoute mit `RequestRedirect` | `http://` leitet mit 301 auf `https://` um |
| Erneuerung | Automatisch durch cert-manager, rund 30 Tage vor Ablauf |

## Ref

  * https://cert-manager.io/docs/usage/gateway/
  * https://cert-manager.io/docs/configuration/acme/http01/#configuring-the-http-01-gateway-api-solver
  * https://letsencrypt.org/docs/staging-environment/
  * https://letsencrypt.org/docs/rate-limits/
