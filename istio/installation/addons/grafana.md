# Installation Grafana (mit HTTPS und Basic Auth ueber Traefik)

## Ziel

Grafana laeuft im Cluster und ist danach unter `https://grafana.<dein-user>.do.t3isp.de/` erreichbar,
geschuetzt mit Benutzer und Passwort.

## Hintergrund

Das Gateway von Istio kann kein Basic Auth. Deshalb laeuft im Cluster zusaetzlich **Traefik** als zweiter
Gateway-API-Controller. Beide arbeiten mit denselben Objekten (`Gateway`, `HTTPRoute`), nur die `GatewayClass` ist eine andere.

| Was | Istio | Traefik |
|-----|-------|---------|
| `GatewayClass` | `istio` | `traefik` |
| Gateway | `bookinfo-gateway` (Namespace `bookinfo`) | `traefik-gateway` (Namespace `traefik`) |
| Aufgabe hier | Verkehr ins Mesh (Bookinfo) | Addons wie Grafana veroeffentlichen |
| Basic Auth | nein | ja, ueber eine `Middleware` |

Die `HTTPRoute` ist Standard-Gateway-API. Nur der Verweis auf die `Middleware` ist eine Erweiterung von Traefik
(Filter `ExtensionRef`) und funktioniert mit keinem anderen Controller.

## Voraussetzung

  * Istio ist installiert (Namespace `istio-system`).
  * Prometheus ist installiert, siehe [Installation Prometheus](prometheus.md). Grafana holt seine Daten von dort.
  * Traefik, DNS und Zertifikat sind schon da (siehe Merkkasten).

> **Merkkasten: Traefik, DNS und Zertifikat sind schon eingerichtet**
>
> Der Trainer hat fuer jeden Cluster vorab eingerichtet:
>
> * Traefik im Namespace `traefik` mit dem Gateway `traefik-gateway` (Listener `web` fuer HTTP, `websecure` fuer HTTPS)
> * den DNS-Eintrag `*.<dein-user>.do.t3isp.de`, z. B. `grafana.tln5.do.t3isp.de`
> * ein Zertifikat von Let's Encrypt fuer `*.<dein-user>.do.t3isp.de`
> * die Weiterleitung von `http://` auf `https://`
>
> Du musst **nichts** im DNS eintragen und kein Zertifikat holen.

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir -p grafana
cd grafana
```

```
export HOST=grafana.$(whoami).do.t3isp.de
echo $HOST $(dig +short $HOST)
```

Erwartet: Hostname und eine IP. Ist die IP leer: Trainer fragen.

```
kubectl get gatewayclass traefik
kubectl -n traefik get gateway traefik-gateway
```

Erwartet: `ACCEPTED True` und `PROGRAMMED True`, `ADDRESS` ist die IP von oben.

## Schritt 2: Grafana installieren

```
kubectl -n istio-system get deployment prometheus
```

Erwartet: `READY 1/1`. Fehlt Prometheus: zuerst [Installation Prometheus](prometheus.md).

```
kubectl apply -f ~/istio/samples/addons/grafana.yaml
kubectl -n istio-system rollout status deployment grafana
```

```
kubectl -n istio-system get pods -l app.kubernetes.io/name=grafana
kubectl -n istio-system get svc grafana
```

Erwartet: 1 Pod `Running`. Der Service `grafana` (Port 3000) ist die Oberflaeche.

Grafana selbst fragt nicht nach einem Login (anonymer Zugang im Addon). Den Schutz uebernimmt Traefik.

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

Secret und `Middleware` sind fuer alle Addons dieselben. Hast du sie schon bei Jaeger, Prometheus oder Kiali angelegt,
meldet `kubectl` hier `unchanged`. Das ist in Ordnung.


## Schritt 4: HTTPRoute anlegen

Die Route haengt am Listener `websecure` (HTTPS) des Traefik-Gateways. Die Regel fragt das Passwort ab
und reicht an Grafana weiter.

```
# vi 03-httproute.yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
spec:
  parentRefs:
  - name: traefik-gateway
    namespace: traefik
    sectionName: websecure
  hostnames:
  - ${HOST}
  rules:
  - filters:
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: basic-auth
    backendRefs:
    - name: grafana
      port: 3000
```

```
envsubst < 03-httproute.yml | kubectl -n istio-system apply -f -
```

```
kubectl -n istio-system get httproute grafana
kubectl -n istio-system get httproute grafana -o jsonpath='{.status.parents[0].conditions[*].type}{"\n"}{.status.parents[0].conditions[*].status}{"\n"}'
```

Erwartet: `Accepted ResolvedRefs` und darunter `True True`.

## Schritt 5: Testen

```
curl -s -o /dev/null -w '%{http_code}\n' https://$HOST/api/health
```

Erwartet: `401` (ohne Benutzer und Passwort kommt niemand durch).

```
curl -s -o /dev/null -w '%{http_code}\n' -u 'training:myS3cr3t!' https://$HOST/api/health
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://$HOST/api/health
```

Erwartet: `200`, danach `301 https://...` (HTTP wird auf HTTPS umgeleitet).

```
curl -s -u 'training:myS3cr3t!' "https://$HOST/api/search?query=Istio" | grep -o '"title":"[^"]*"'
```

Erwartet: die Dashboards von Istio, z. B. `Istio Mesh Dashboard` und `Istio Service Dashboard`.

```
echo https://$HOST/
```

Die URL im Browser oeffnen und mit `training` / `myS3cr3t!` anmelden: Die Oberflaeche von Grafana erscheint,
Schloss in der Adressleiste, Aussteller `Let's Encrypt`.

Die Dashboards siehst du dir in der Uebung [Grafana Dashboards fuer istio](../../exercises/11-grafana.md) an.

## Fehlersuche

| Beobachtung | Pruefen |
|-------------|---------|
| `404` statt `401` | Stimmt der Hostname der Route? `kubectl -n istio-system get httproute grafana` |
| `401` trotz richtigem Passwort | Secret hat den Schluessel `users` (nicht `auth` wie bei ingress-nginx) |
| `500` oder `404` mit Passwort | Gibt es die `Middleware` im selben Namespace wie die Route? `kubectl -n istio-system get middleware` |
| Hostname wird nicht gefunden | `dig +short $HOST`, sonst Trainer fragen |
| Dashboards sind leer | Laeuft Prometheus? `kubectl -n istio-system get deployment prometheus` |

## Ref

  * https://istio.io/latest/docs/ops/integrations/grafana/
  * https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/
  * https://gateway-api.sigs.k8s.io/api-types/httproute/
