# Uebung: Distributed Tracing mit Jaeger (Ambient)

## Ziel

Istio schickt Traces an Jaeger. Du siehst in Jaeger den Weg einer Anfrage an Bookinfo:
vom Gateway zu `productpage` und von dort ueber den Waypoint zu `details`, `reviews` und `ratings`.

## Voraussetzung

  * [Installation Jaeger](../installation/addons/jaeger.md) ist durchgefuehrt
    (Jaeger laeuft, `https://jaeger.<dein-user>.do.t3isp.de/jaeger/` fragt nach Benutzer und Passwort).
  * [Bookinfo demo in ambient ausrollen](../installation/ambient/04-install-demo-app.md) ist durchgefuehrt
    (Namespace `bookinfo` mit Waypoint, Gateway `bookinfo-gateway`).
  * Istio ist mit `istioctl install --set profile=ambient` installiert.

## Hintergrund

Zwei Dinge fehlen nach der Installation noch:

| Was | Womit | Wirkung |
|-----|-------|---------|
| Istio kennt Jaeger nicht | `meshConfig.extensionProviders` (Datei fuer `istioctl install -f`) | Provider `jaeger` zeigt auf `jaeger-collector`, Port 4317 (OpenTelemetry) |
| Niemand nutzt den Provider | `Telemetry` in `istio-system` | Tracing fuer das ganze Mesh einschalten |

Im Ambient-Mode schreiben nur **Gateway** und **Waypoint** Spans, weil nur sie HTTP verstehen (Layer 7).
Der `ztunnel` arbeitet auf Layer 4 und erzeugt keine Spans. Ohne Waypoint gibt es deshalb nur den Span vom Gateway.

## Schritt 1: Vorbereitung

```
cd
mkdir -p manifests
cd manifests
mkdir tracing-jaeger
cd tracing-jaeger
```

```
export HOST=jaeger.$(whoami).do.t3isp.de
export GATEWAY_IP=$(kubectl -n bookinfo get gateway bookinfo-gateway -o jsonpath='{.status.addresses[0].value}')
echo $HOST $GATEWAY_IP
```

Beide Werte muessen gefuellt sein.

## Schritt 2: Istio den Provider Jaeger mitgeben

Vorher: Es gibt noch keinen Provider.

```
kubectl -n istio-system get cm istio -o jsonpath='{.data.mesh}' | grep -A4 extensionProviders
```

Erwartet: keine Ausgabe.

Die Einstellung kommt in eine Datei. `kind: IstioOperator` ist dabei nur das Dateiformat, das `istioctl install` liest.
Im Cluster laeuft **kein** Operator (den gibt es in aktuellen Istio-Versionen nicht mehr), und die Datei wird auch nicht mit `kubectl apply` angewendet.

```
# vi 01-istio-tracing.yml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: ambient
  meshConfig:
    extensionProviders:
    - name: jaeger
      opentelemetry:
        service: jaeger-collector.istio-system.svc.cluster.local
        port: 4317
  values:
    pilot:
      traceSampling: 100
```

```
istioctl install -f 01-istio-tracing.yml -y
```

`traceSampling: 100` zeichnet jede Anfrage auf (Standard: 1 von 100). Gut zum Ueben, in Produktion zu viel.

```
kubectl -n istio-system rollout status deployment istiod
kubectl -n istio-system get cm istio -o jsonpath='{.data.mesh}' | grep -A4 extensionProviders
```

Erwartet:

```
extensionProviders:
- name: jaeger
  opentelemetry:
    port: 4317
    service: jaeger-collector.istio-system.svc.cluster.local
```

## Schritt 3: Tracing einschalten

```
# vi 02-telemetry-jaeger.yml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
spec:
  tracing:
  - providers:
    - name: jaeger
```

```
kubectl -n istio-system apply -f 02-telemetry-jaeger.yml
kubectl -n istio-system get telemetry
```

Ein `Telemetry` im Namespace `istio-system` gilt fuer das ganze Mesh.

## Schritt 4: Anfragen erzeugen

```
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_IP/productpage"; done
curl -s -o /dev/null -w '%{http_code}\n' "http://$GATEWAY_IP/productpage"
```

Erwartet: `200`.

Hast du die Uebung mit HTTPS und Let's Encrypt gemacht, antwortet das Gateway nur noch unter seinem Namen.
Dann stattdessen:

```
# nur nach der Uebung HTTPS mit Let's Encrypt
# for i in $(seq 1 100); do curl -s -o /dev/null "https://bookinfo.$(whoami).do.t3isp.de/productpage"; done
```

## Schritt 5: Traces in Jaeger ansehen

Zuerst auf der Kommandozeile: Welche Services kennt Jaeger?

```
sleep 10
curl -s -u 'training:myS3cr3t!' https://$HOST/jaeger/api/services
```

Erwartet: `bookinfo-gateway-istio.bookinfo` und `waypoint.bookinfo` (dazu `jaeger` selbst).

```
echo https://$HOST/jaeger/
```

Die URL im Browser oeffnen (Benutzer `training`, Passwort `myS3cr3t!`):

```
# Links unter "Service": bookinfo-gateway-istio.bookinfo auswaehlen
# Ganz unten: "Find Traces" anklicken
# Oben einen Punkt bzw. einen Trace anklicken
```

<img width="436" height="147" alt="image" src="https://github.com/user-attachments/assets/0bb5069b-ee3c-4799-91fa-35b498a3c047" />

<img width="444" height="61" alt="image" src="https://github.com/user-attachments/assets/97451972-7070-46ab-a4e4-39dc144f60b1" />

Ein Trace von `/productpage` hat 5 Spans:

| Wer schreibt den Span | Ziel |
|-----------------------|------|
| `bookinfo-gateway-istio.bookinfo` | `productpage` (2 Spans: Anfrage und Weiterleitung) |
| `waypoint.bookinfo` | `details` |
| `waypoint.bookinfo` | `reviews` |
| `waypoint.bookinfo` | `ratings` |

## Fragen

  1. Warum taucht `productpage` nicht als eigener Service in Jaeger auf?
  2. Was siehst du in Jaeger, wenn der Namespace keinen Waypoint hat?
  3. Warum haengen die Spans von Gateway und Waypoint in **einem** Trace zusammen?

## Loesungen

  1. Im Ambient-Mode hat `productpage` keinen Sidecar. Spans schreiben nur Gateway und Waypoint.
  2. Nur noch die Spans vom Gateway. Der `ztunnel` (Layer 4) schreibt keine.
  3. Bookinfo reicht die Trace-Header der eingehenden Anfrage (z. B. `traceparent`, `x-b3-traceid`) an seine eigenen Aufrufe weiter.
     Ohne das waeren es einzelne, unverbundene Traces. Das muss die Anwendung tun, Istio kann es nicht fuer sie.

## Aufraeumen

```
kubectl -n istio-system delete -f 02-telemetry-jaeger.yml
```

Istio wieder ohne Provider und mit dem Standard-Sampling:

```
istioctl install --set profile=ambient -y
kubectl -n istio-system rollout status deployment istiod
kubectl -n istio-system get cm istio -o jsonpath='{.data.mesh}' | grep -A4 extensionProviders
```

Erwartet: keine Ausgabe mehr.

## Ref

  * https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/
  * https://istio.io/latest/docs/tasks/observability/telemetry/
