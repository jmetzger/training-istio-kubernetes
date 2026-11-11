# Installation Kiali (mit HTTPS und Basic Auth ueber Traefik)

## Ziel

Kiali laeuft im Cluster und ist danach unter `https://kiali.<dein-user>.do.t3isp.de/kiali/` erreichbar,
geschuetzt mit Benutzer und Passwort.

## Hintergrund

Das Gateway von Istio kann kein Basic Auth. Deshalb laeuft im Cluster zusaetzlich **Traefik** als zweiter
Gateway-API-Controller. Beide arbeiten mit denselben Objekten (`Gateway`, `HTTPRoute`), nur die `GatewayClass` ist eine andere.

| Was | Istio | Traefik |
|-----|-------|---------|
| `GatewayClass` | `istio` | `traefik` |
| Gateway | `bookinfo-gateway` (Namespace `bookinfo`) | `traefik-gateway` (Namespace `traefik`) |
| Aufgabe hier | Verkehr ins Mesh (Bookinfo) | Addons wie Kiali veroeffentlichen |
| Basic Auth | nein | ja, ueber eine `Middleware` |

Die `HTTPRoute` ist Standard-Gateway-API. Nur der Verweis auf die `Middleware` ist eine Erweiterung von Traefik
(Filter `ExtensionRef`) und funktioniert mit keinem anderen Controller.

## Voraussetzung

  * Istio ist installiert (Namespace `istio-system`).
  * Prometheus ist installiert, siehe [Installation Prometheus](prometheus.md). Ohne Prometheus startet Kiali, zeigt aber keinen Graphen.
  * Traefik, DNS und Zertifikat sind schon da (siehe Merkkasten).

> **Merkkasten: Traefik, DNS und Zertifikat sind schon eingerichtet**
>
> Der Trainer hat fuer jeden Cluster vorab eingerichtet:
>
> * Traefik im Namespace `traefik` mit dem Gateway `traefik-gateway` (Listener `web` fuer HTTP, `websecure` fuer HTTPS)
> * den DNS-Eintrag `*.<dein-user>.do.t3isp.de`, z. B. `kiali.tln5.do.t3isp.de`
> * ein Zertifikat von Let's Encrypt fuer `*.<dein-user>.do.t3isp.de`
> * die Weiterleitung von `http://` auf `https://`
>
> Du musst **nichts** im DNS eintragen und kein Zertifikat holen.

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir -p kiali
cd kiali
```

```
export HOST=kiali.$(whoami).do.t3isp.de
echo $HOST $(dig +short $HOST)
```

Erwartet: Hostname und eine IP. Ist die IP leer: Trainer fragen.

```
kubectl get gatewayclass traefik
kubectl -n traefik get gateway traefik-gateway
```

Erwartet: `ACCEPTED True` und `PROGRAMMED True`, `ADDRESS` ist die IP von oben.

## Schritt 2: Kiali installieren

```
kubectl -n istio-system get deployment prometheus
```

Erwartet: `READY 1/1`. Fehlt Prometheus: zuerst [Installation Prometheus](prometheus.md).

```
kubectl apply -f ~/istio/samples/addons/kiali.yaml
kubectl -n istio-system rollout status deployment kiali
```

```
kubectl -n istio-system get pods -l app.kubernetes.io/name=kiali
kubectl -n istio-system get svc kiali
```

Erwartet: 1 Pod `Running`. Der Service `kiali` (Port 20001) ist die Oberflaeche.

Kiali selbst fragt nicht nach einem Login (`auth.strategy: anonymous` im Addon). Den Schutz uebernimmt Traefik.

## Schritt 3: Benutzer und Passwort (Basic Auth)

Benutzer `training`, Passwort `myS3cr3t!`. Im Secret steht nicht das Passwort, sondern eine htpasswd-Zeile (bcrypt).

```
# vi 01-basic-auth-secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: addons-basic-auth
type: Opaque
stringData:
  users: "training:$2b$12$CfOZaJ.Tr0zu6PfpbuCjzeKiQ2PzZARfP.CbC6tRU/7OvEHCIOREm"
```

```
# vi 02-middleware.yml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
spec:
  basicAuth:
    secret: addons-basic-auth
    realm: Training
```

```
kubectl -n istio-system apply -f 01-basic-auth-secret.yml -f 02-middleware.yml
```

Secret und `Middleware` sind fuer alle Addons dieselben. Hast du sie schon bei Jaeger, Prometheus oder Grafana angelegt,
meldet `kubectl` hier `unchanged`. Das ist in Ordnung.


## Schritt 4: HTTPRoute anlegen

Die Route haengt am Listener `websecure` (HTTPS) des Traefik-Gateways. Kiali laeuft unter dem Pfad `/kiali`,
deshalb leitet die erste Regel `/` dorthin weiter. Die zweite Regel fragt das Passwort ab und reicht an Kiali weiter.

```
# vi 03-httproute.yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kiali
spec:
  parentRefs:
  - name: traefik-gateway
    namespace: traefik
    sectionName: websecure
  hostnames:
  - ${HOST}
  rules:
  - matches:
    - path:
        type: Exact
        value: /
    filters:
    - type: RequestRedirect
      requestRedirect:
        path:
          type: ReplaceFullPath
          replaceFullPath: /kiali/
        statusCode: 302
  - filters:
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: basic-auth
    backendRefs:
    - name: kiali
      port: 20001
```

```
envsubst < 03-httproute.yml | kubectl -n istio-system apply -f -
```

```
kubectl -n istio-system get httproute kiali
kubectl -n istio-system get httproute kiali -o jsonpath='{.status.parents[0].conditions[*].type}{"\n"}{.status.parents[0].conditions[*].status}{"\n"}'
```

Erwartet: `Accepted ResolvedRefs` und darunter `True True`.

## Schritt 5: Testen

```
curl -s -o /dev/null -w '%{http_code}\n' https://$HOST/kiali/
```

Erwartet: `401` (ohne Benutzer und Passwort kommt niemand durch).

```
curl -s -o /dev/null -w '%{http_code}\n' -u 'training:myS3cr3t!' https://$HOST/kiali/
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://$HOST/kiali/
```

Erwartet: `200`, danach `301 https://...` (HTTP wird auf HTTPS umgeleitet).

```
curl -s -u 'training:myS3cr3t!' https://$HOST/kiali/api/istio/status
```

Erwartet: eine Liste mit `istiod` und `prometheus`, beide mit `"status":"Healthy"`.

```
echo https://$HOST/kiali/
```

Die URL im Browser oeffnen und mit `training` / `myS3cr3t!` anmelden: Die Oberflaeche von Kiali erscheint,
Schloss in der Adressleiste, Aussteller `Let's Encrypt`.

Den Graphen mit Verkehr siehst du in der Uebung [Visualisierung mit Kiali](../../exercises/05-kiali-traffic.md).

## Fehlersuche

| Beobachtung | Pruefen |
|-------------|---------|
| `404` statt `401` | Stimmt der Hostname der Route? `kubectl -n istio-system get httproute kiali` |
| `401` trotz richtigem Passwort | Secret hat den Schluessel `users` (nicht `auth` wie bei ingress-nginx) |
| `500` oder `404` mit Passwort | Gibt es die `Middleware` im selben Namespace wie die Route? `kubectl -n istio-system get middleware` |
| Hostname wird nicht gefunden | `dig +short $HOST`, sonst Trainer fragen |
| Kiali meldet `Could not fetch ... Prometheus` | Laeuft Prometheus? `kubectl -n istio-system get deployment prometheus` |

## Ref

  * https://istio.io/latest/docs/ops/integrations/kiali/
  * https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/
  * https://gateway-api.sigs.k8s.io/api-types/httproute/
