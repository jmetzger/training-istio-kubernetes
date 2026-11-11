# Installation Prometheus (mit HTTPS und Basic Auth ueber Traefik)

## Ziel

Prometheus laeuft im Cluster und ist danach unter `https://prometheus.<dein-user>.do.t3isp.de/` erreichbar,
geschuetzt mit Benutzer und Passwort.

## Hintergrund

Das Gateway von Istio kann kein Basic Auth. Deshalb laeuft im Cluster zusaetzlich **Traefik** als zweiter
Gateway-API-Controller. Beide arbeiten mit denselben Objekten (`Gateway`, `HTTPRoute`), nur die `GatewayClass` ist eine andere.

| Was | Istio | Traefik |
|-----|-------|---------|
| `GatewayClass` | `istio` | `traefik` |
| Gateway | `bookinfo-gateway` (Namespace `bookinfo`) | `traefik-gateway` (Namespace `traefik`) |
| Aufgabe hier | Verkehr ins Mesh (Bookinfo) | Addons wie Prometheus veroeffentlichen |
| Basic Auth | nein | ja, ueber eine `Middleware` |

Die `HTTPRoute` ist Standard-Gateway-API. Nur der Verweis auf die `Middleware` ist eine Erweiterung von Traefik
(Filter `ExtensionRef`) und funktioniert mit keinem anderen Controller.

## Voraussetzung

  * Istio ist installiert (Namespace `istio-system`).
  * Traefik, DNS und Zertifikat sind schon da (siehe Merkkasten).

> **Merkkasten: Traefik, DNS und Zertifikat sind schon eingerichtet**
>
> Der Trainer hat fuer jeden Cluster vorab eingerichtet:
>
> * Traefik im Namespace `traefik` mit dem Gateway `traefik-gateway` (Listener `web` fuer HTTP, `websecure` fuer HTTPS)
> * den DNS-Eintrag `*.<dein-user>.do.t3isp.de`, z. B. `prometheus.tln5.do.t3isp.de`
> * ein Zertifikat von Let's Encrypt fuer `*.<dein-user>.do.t3isp.de`
> * die Weiterleitung von `http://` auf `https://`
>
> Du musst **nichts** im DNS eintragen und kein Zertifikat holen.

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir -p prometheus
cd prometheus
```

```
export HOST=prometheus.$(whoami).do.t3isp.de
echo $HOST $(dig +short $HOST)
```

Erwartet: Hostname und eine IP. Ist die IP leer: Trainer fragen.

```
kubectl get gatewayclass traefik
kubectl -n traefik get gateway traefik-gateway
```

Erwartet: `ACCEPTED True` und `PROGRAMMED True`, `ADDRESS` ist die IP von oben.

## Schritt 2: Prometheus installieren

```
kubectl apply -f ~/istio/samples/addons/prometheus.yaml
kubectl -n istio-system rollout status deployment prometheus
```

```
kubectl -n istio-system get pods -l app.kubernetes.io/name=prometheus
kubectl -n istio-system get svc prometheus
```

Erwartet: 1 Pod `Running` (`2/2`). Der Service `prometheus` (Port 9090) ist Oberflaeche und API zugleich.

Prometheus hat von sich aus keinen Login. Den Schutz uebernimmt Traefik.

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

Secret und `Middleware` sind fuer alle Addons dieselben. Hast du sie schon bei Jaeger, Grafana oder Kiali angelegt,
meldet `kubectl` hier `configured` oder `unchanged`. Beides ist in Ordnung.

## Schritt 4: HTTPRoute anlegen

Die Route haengt am Listener `websecure` (HTTPS) des Traefik-Gateways. Die Regel fragt das Passwort ab
und reicht an Prometheus weiter.

```
# vi 03-httproute.yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: prometheus
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
    - name: prometheus
      port: 9090
```

```
envsubst < 03-httproute.yml | kubectl -n istio-system apply -f -
```

```
kubectl -n istio-system get httproute prometheus
kubectl -n istio-system get httproute prometheus -o jsonpath='{.status.parents[0].conditions[*].type}{"\n"}{.status.parents[0].conditions[*].status}{"\n"}'
```

Erwartet: `Accepted ResolvedRefs` und darunter `True True`.

## Schritt 5: Testen

```
curl -s -o /dev/null -w '%{http_code}\n' https://$HOST/-/ready
```

Erwartet: `401` (ohne Benutzer und Passwort kommt niemand durch).

```
curl -s -o /dev/null -w '%{http_code}\n' -u 'training:myS3cr3t!' https://$HOST/-/ready
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://$HOST/-/ready
```

Erwartet: `200`, danach `301 https://...` (HTTP wird auf HTTPS umgeleitet).

```
curl -s -u 'training:myS3cr3t!' "https://$HOST/api/v1/query?query=count(up)"
```

Erwartet: `"status":"success"` und eine Zahl groesser 0 (so viele Ziele fragt Prometheus ab).
Ist `result` noch leer (`[]`): 30 Sekunden warten und wiederholen, Prometheus hat dann noch nichts abgefragt.

```
echo https://$HOST/
```

Die URL im Browser oeffnen und mit `training` / `myS3cr3t!` anmelden: Die Oberflaeche von Prometheus erscheint,
Schloss in der Adressleiste, Aussteller `Let's Encrypt`.

Abfragen zu Istio machst du in der Uebung [Metriken mit Prometheus auswerten](../../exercises/10-prometheus.md).

## Fehlersuche

| Beobachtung | Pruefen |
|-------------|---------|
| `404` statt `401` | Stimmt der Hostname der Route? `kubectl -n istio-system get httproute prometheus` |
| `401` trotz richtigem Passwort | Secret hat den Schluessel `users` (nicht `auth` wie bei ingress-nginx) |
| `500` oder `404` mit Passwort | Gibt es die `Middleware` im selben Namespace wie die Route? `kubectl -n istio-system get middleware` |
| Hostname wird nicht gefunden | `dig +short $HOST`, sonst Trainer fragen |

## Ref

  * https://istio.io/latest/docs/ops/integrations/prometheus/
  * https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/
  * https://gateway-api.sigs.k8s.io/api-types/httproute/
