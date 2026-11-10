# Uebung: Egress-Gateway im Ambient-Mesh (Waypoint als Egress-Gateway)

## Ziel

Pods im Mesh sollen externe Dienste (hier `httpbin.org`) nur noch **ueber einen zentralen Punkt** erreichen.
An diesem Punkt pruefen wir Methode und Pfad, bauen TLS zum Ziel auf und koennen mitlesen.
Im Sidecar-Mode ist das ein eigener Egress-Gateway-Pod. **In Ambient uebernimmt diese Rolle ein Waypoint.**

![Egress in Ambient: Waypoint als Egress-Gateway](../../images/istio/egress-waypoint-ambient.svg)

## Voraussetzung

  * Istio Ambient ist installiert, siehe [istio-installation mit istioctl - ambient](../installation/ambient/03-install-with-istioctl-ambient.md).
    Bookinfo wird fuer diese Uebung **nicht** gebraucht.
  * Die Gateway-API-CRDs sind da (sie werden fuer jeden Waypoint gebraucht):

```
kubectl get crd gateways.gateway.networking.k8s.io
```

  * Das Cluster darf ins Internet (`httpbin.org`, `example.com`).

## Hintergrund

| Frage | Antwort in Ambient |
|---|---|
| Wo laeuft der Egress-Gateway? | Als Waypoint in einem eigenen Namespace, hier `istio-egress` |
| Wie kommt der Verkehr dorthin? | Eine `ServiceEntry` im Namespace des Waypoints beschreibt das Ziel. Der Namespace traegt das Label `istio.io/use-waypoint`, damit gehen alle Ziele darin ueber den Waypoint. |
| Wer schaut auf Methode und Pfad? | Der Waypoint (Layer 7), per `AuthorizationPolicy` auf die `ServiceEntry` |
| Wer baut TLS zum Ziel auf? | Der Waypoint, per `DestinationRule`. Die Anwendung spricht weiter HTTP. |
| Wer erzwingt den Weg? | **Nicht der Waypoint.** Das macht eine `NetworkPolicy` (Schritt 8). |

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir egress-ambient
cd egress-ambient
```

## Schritt 2: Client im Mesh anlegen und Ausgangslage pruefen

Ein Namespace im Ambient-Mesh mit einem kleinen curl-Pod:

```
kubectl create namespace egress-demo
kubectl label namespace egress-demo istio.io/dataplane-mode=ambient
kubectl -n egress-demo apply -f ~/istio/samples/curl/curl.yaml
kubectl -n egress-demo rollout status deploy/curl
```

Zugriff ins Internet, so wie es heute ist:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -o /dev/null -w "httpbin.org -> %{http_code}\n" http://httpbin.org/get
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -o /dev/null -w "example.com -> %{http_code}\n" http://example.com/
```

Erwartet:

```
httpbin.org -> 200
example.com -> 200
```

Ausgangslage: Der Pod darf **alles** erreichen. Es gibt keinen zentralen Punkt, an dem wir pruefen oder mitlesen koennten.

## Schritt 3: Waypoint als Egress-Gateway anlegen

Eigener Namespace fuer den Egress-Gateway, darin ein Waypoint. `--enroll-namespace` setzt das Label `istio.io/use-waypoint`
auf den Namespace:

```
kubectl create namespace istio-egress
istioctl waypoint apply --for service --enroll-namespace -n istio-egress --wait
```

Erwartet (die Warnung zu `dataplane-mode` kannst du ignorieren, der Waypoint selbst gehoert nicht ins Mesh):

```
✅ waypoint istio-egress/waypoint is ready!
✅ namespace istio-egress labeled with "istio.io/use-waypoint: waypoint"
```

```
istioctl waypoint list -n istio-egress
kubectl get namespace istio-egress --show-labels
```

`PROGRAMMED` muss `True` sein.

## Schritt 4: Externen Dienst als ServiceEntry beschreiben

```
# vi 01-serviceentry-httpbin.yml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
```

```
kubectl apply -f 01-serviceentry-httpbin.yml -n istio-egress
```

Aufruf wie vorher:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -o /dev/null -w "httpbin.org -> %{http_code}\n" http://httpbin.org/get
```

Erwartet: `httpbin.org -> 200`. Fuer die Anwendung hat sich nichts geaendert. Aber der Weg ist jetzt ein anderer.
Das sehen wir im ztunnel-Log auf dem Node des curl-Pods:

```
NODE=$(kubectl -n egress-demo get pod -l app=curl -o jsonpath="{.items[0].spec.nodeName}")
ZT=$(kubectl -n istio-system get pod -l app=ztunnel --field-selector spec.nodeName=$NODE -o name)
kubectl -n istio-system logs $ZT --tail=200 | grep egress-demo | grep httpbin | tail -2
```

Erwartet (gekuerzt):

```
src.workload="curl-..." src.namespace="egress-demo" ... dst.addr=192.168.x.x:15008 dst.service="httpbin.org"
dst.workload="waypoint-..." dst.namespace="istio-egress" ... direction="outbound"
```

Das Ziel ist der **Waypoint auf Port 15008** (HBONE), nicht `httpbin.org`. Der Waypoint macht von dort aus den Aufruf ins Internet.

Noch ein Hinweis, wer nach draussen auftritt:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 http://httpbin.org/get
```

Im Feld `origin` steht die oeffentliche IP des **Nodes, auf dem der Waypoint laeuft**. Beim Zielsystem kommen alle Aufrufe
von dort an, egal von welchem Pod sie stammen.

## Schritt 5: Nur GET auf /get erlauben

Layer 7 ist jetzt moeglich, weil der Waypoint im Weg ist. Die Policy haengt an der `ServiceEntry`:

```
# vi 02-authorizationpolicy-httpbin.yml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: httpbin-nur-get
spec:
  targetRefs:
  - kind: ServiceEntry
    group: networking.istio.io
    name: httpbin
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
        paths: ["/get"]
```

```
kubectl apply -f 02-authorizationpolicy-httpbin.yml -n istio-egress
```

Ein paar Sekunden warten, dann drei Aufrufe:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -X GET  -o /dev/null -w "GET /get      -> %{http_code}\n" http://httpbin.org/get
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -X GET  -o /dev/null -w "GET /headers  -> %{http_code}\n" http://httpbin.org/headers
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -X POST -o /dev/null -w "POST /post    -> %{http_code}\n" http://httpbin.org/post
```

Erwartet:

```
GET /get      -> 200
GET /headers  -> 403
POST /post    -> 403
```

**Erwarteter Fehler** bei einem abgelehnten Aufruf:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -X POST http://httpbin.org/post
```

```
RBAC: access denied
```

Die Ablehnung kommt vom Waypoint. Den Zaehler dazu siehst du so:

```
kubectl -n istio-egress exec deploy/waypoint -- pilot-agent request GET stats | grep -E "rbac\.(allowed|denied)"
```

Erwartet: `rbac.allowed: 1` und `rbac.denied: 2` oder mehr (die Zahlen haengen davon ab, wie oft du aufgerufen hast).

## Schritt 6: TLS zum Ziel am Waypoint aufbauen (TLS-Origination)

Die Anwendung bleibt bei HTTP auf Port 80. Der Waypoint macht daraus HTTPS auf Port 443.
`targetPort` in der `ServiceEntry` und eine `DestinationRule` reichen dafuer:

```
# vi 03-serviceentry-httpbin-tls.yml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
    targetPort: 443
  resolution: DNS
```

```
# vi 04-destinationrule-httpbin-tls.yml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin-tls
spec:
  host: httpbin.org
  trafficPolicy:
    tls:
      mode: SIMPLE
```

```
kubectl apply -f 03-serviceentry-httpbin-tls.yml -n istio-egress
kubectl apply -f 04-destinationrule-httpbin-tls.yml -n istio-egress
```

Ein paar Sekunden warten, dann:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 http://httpbin.org/get
```

Erwartet (gekuerzt):

```
  "url": "https://httpbin.org/get"
```

Wir haben `http://` aufgerufen, beim Ziel kam aber `https://` an. Das TLS hat der Waypoint aufgebaut.
Die Anwendung muss dafuer nichts wissen, und Zertifikatsthemen fuer externe Ziele liegen an **einer** Stelle.

## Schritt 7: Die Luecke zeigen - ein Ziel ohne ServiceEntry

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 15 -o /dev/null -w "example.com -> %{http_code}\n" http://example.com/
```

Erwartet: `example.com -> 200`.

`example.com` hat keine `ServiceEntry`. Der Aufruf geht **am Waypoint vorbei direkt ins Internet**.
Auch eine Anwendung, die absichtlich eine andere Route nimmt, kommt so raus. Die Einbindung des Namespaces in den Waypoint
**lenkt** den Verkehr, sie **erzwingt** ihn nicht.

## Schritt 8: Den Weg erzwingen mit einer NetworkPolicy

Der Pod darf nur noch zu zwei Zielen: DNS und Waypoint (Port 15008, HBONE). Alles andere wird vom CNI (hier Calico) verworfen.

```
# vi 05-networkpolicy-egress.yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-nur-ueber-waypoint
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: istio-egress
      podSelector:
        matchLabels:
          gateway.networking.k8s.io/gateway-name: waypoint
    ports:
    - protocol: TCP
      port: 15008
```

```
kubectl apply -f 05-networkpolicy-egress.yml -n egress-demo
```

Ein paar Sekunden warten, dann beide Ziele:

```
kubectl -n egress-demo exec deploy/curl -- curl -s -m 10 -o /dev/null -w "httpbin.org -> %{http_code}\n" http://httpbin.org/get
kubectl -n egress-demo exec deploy/curl -- curl -s -m 10 -o /dev/null -w "example.com -> %{http_code}\n" http://example.com/
```

Erwartet:

```
httpbin.org -> 200
example.com -> 000
command terminated with exit code 28
```

`httpbin.org` geht weiter, weil der Weg ueber den Waypoint erlaubt ist. `example.com` laeuft nach 10 Sekunden in den Timeout
(curl-Exit-Code 28), weil der direkte Weg gesperrt ist.

Soll `example.com` auch erlaubt werden, ist der Weg immer derselbe: eine `ServiceEntry` im Namespace `istio-egress` anlegen,
und, wenn noetig, eine `AuthorizationPolicy` dazu. Eine Anwendung kann sich den Weg nicht selbst aufmachen.

## Aufraeumen

Beide Namespaces enthalten alles, was wir angelegt haben (Waypoint, ServiceEntry, Policies, curl-Pod):

```
kubectl delete namespace egress-demo istio-egress
```

```
kubectl get namespace | grep egress
```

Erwartet: keine Ausgabe.

```
cd ~/manifests
rm -rf egress-ambient
```

## Zusammenfassung

| Schritt | Was wir gemacht haben | Ergebnis |
|---|---|---|
| 2 | Pod im Mesh, kein Egress-Gateway | `httpbin.org` und `example.com` erreichbar |
| 4 | ServiceEntry im Namespace mit Waypoint | 200, ztunnel-Log zeigt Waypoint `:15008` als Ziel |
| 5 | AuthorizationPolicy auf die ServiceEntry | `GET /get` 200, `GET /headers` und `POST /post` 403 |
| 6 | `targetPort` 443 und DestinationRule | HTTP in der App, HTTPS zum Ziel |
| 7 | Ziel ohne ServiceEntry | 200, der Waypoint wird umgangen |
| 8 | NetworkPolicy: nur DNS und Waypoint | `httpbin.org` 200, `example.com` Timeout |

Merksatz: **Der Waypoint steuert den Weg und die Regeln (Layer 7, TLS). Erzwingen muss ihn eine NetworkPolicy.**

## Referenz

  * [Istio: Egress gateways (Ambient)](https://istio.io/latest/docs/ambient/usage/egress-gateway/)
  * [Istio: Simplifying Egress Routing to Wildcard Destinations](https://istio.io/latest/blog/2026/egress-dynamic-dns/) (Wildcard-Ziele wie `*.amazonaws.com`, ServiceEntry mit `istio.io/use-waypoint` und `DYNAMIC_DNS`)
