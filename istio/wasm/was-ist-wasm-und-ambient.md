# Wasm in Istio: Was ist das, wofür brauche ich es, geht es mit Ambient?

**Stand: 11.11.2026.** Grundlage ist die Istio-Doku zu Version 1.31 (unser Stand: 1.31.4).
Das Beispiel unten stammt aus der Doku und ist **auf dem Trainings-Cluster nicht ausgeführt**.
Geprüft ist nur: Die CRD `TrafficExtension` ist auf unserem Cluster vorhanden.

## Kurzfassung

* **Wasm** (WebAssembly) ist ein Binärformat. Du schreibst eine Erweiterung in Go, Rust oder C++, übersetzt sie in ein `.wasm`-Modul, und Envoy führt sie in einer Sandbox aus.
* **Wofür:** für Logik im Datenpfad, die Istio nicht mitbringt, ohne Envoy neu zu bauen und ohne die Anwendung zu ändern.
* **Ambient:** ja, aber **nur am Waypoint** (oder am Gateway). Im ztunnel läuft kein Wasm. Das Feature ist **Alpha**.

## Was ist Wasm?

* Ursprünglich für den Browser gebaut: Code aus vielen Sprachen läuft dort in einer Sandbox, fast so schnell wie nativ.
* Envoy hat dieselbe Idee übernommen. In Envoy steckt eine Wasm-Laufzeit (V8).
* Die Schnittstelle zwischen Envoy und dem Modul heißt **Proxy-Wasm**. Das Modul bekommt Callbacks wie "Request-Header sind da" und darf Header lesen, ändern, den Request ablehnen oder selbst einen HTTP- oder gRPC-Aufruf machen.
* Das Modul liegt als OCI-Image in einer Registry (oder unter einer HTTP-URL). Der Proxy lädt es zur Laufzeit, ein Neustart ist nicht nötig.

## Wofür brauche ich das in Istio?

Erst prüfen, ob Istio es schon kann. Meistens reicht das:

| Aufgabe | Mittel in Istio |
|---|---|
| Wer darf wen aufrufen | `AuthorizationPolicy` |
| JWT prüfen | `RequestAuthentication` |
| Routing, Retries, Timeouts, Header setzen | `HTTPRoute` |
| Eigener Auth-Dienst entscheidet | `AuthorizationPolicy` mit `action: CUSTOM` (ext_authz) |

Wasm ist für den Rest: eigene Logik, die für jeden Request im Proxy laufen soll. Typische Fälle:

* Anmeldung, die Istio nicht kennt (Basic Auth, API-Key, firmeneigenes Token-Format)
* Rate Limiting mit eigener Regel (siehe [Übung im Backlog](../exercises/16-create-wasm-plugin-ratelimit.md))
* Header oder Body umschreiben, zum Beispiel personenbezogene Daten maskieren
* Eigene Metriken oder Logzeilen aus dem Request ableiten

### Wasm oder Lua?

Istio kennt zwei Arten von Erweiterungen. Beide werden mit `TrafficExtension` eingehängt.

| | Wasm | Lua |
|---|---|---|
| Sprache | Go, Rust, C++ und andere | nur Lua |
| Auslieferung | Modul aus der Registry | Skript steht direkt im Objekt |
| Speicher | höher, jedes Plugin hat seine Sandbox | etwa 10x niedriger |
| Absturz | bleibt im Plugin | kann den Worker-Thread mitreißen |
| Verhalten bei Fehler | einstellbar, Standard ist "Request ablehnen" | immer "Request durchlassen" |
| Passt für | größere Logik, getestet und versioniert | kleine Änderung, Übergangslösung |

## Geht das mit Ambient?

Ja, mit drei Einschränkungen.

![Wasm in Ambient läuft im Waypoint](../../images/istio/wasm-ambient-waypoint.svg)

* **Nur am Waypoint oder Gateway.** Der ztunnel ist kein Envoy und arbeitet nur auf Layer 4. Ein Namespace ohne Waypoint hat keine Erweiterungen.
* **`targetRefs` statt `selector`.** Die `TrafficExtension` zeigt auf den Waypoint (alle Services dahinter), auf einen einzelnen Service oder auf ein Gateway. Ein `selector` wird am Waypoint ignoriert.
* **Alpha.** Für Entwickler und erfahrene Anwender gedacht, die Schnittstelle kann sich ändern.

Dazu die Falle aus dem Mischbetrieb: Verkehr, der den Waypoint umgeht, läuft auch an der Erweiterung vorbei.
Das betrifft Sidecar-Clients und Gateways ohne `ingress-use-waypoint`
(siehe [Was Ambient (noch) nicht kann](../overview/11-ambient-grenzen-gegenueber-sidecar.md) und [Übung: Gateway umgeht den Waypoint](../exercises/24-gateway-umgeht-waypoint-logs.md)).

### `WasmPlugin` oder `TrafficExtension`?

* `WasmPlugin` ist das ältere Objekt, nur für Wasm.
* `TrafficExtension` ist das neue Objekt für Wasm und Lua. Die Referenz schreibt: "This API supersedes WasmPlugin".
* Beide CRDs sind in 1.31.4 vorhanden. Die Ambient-Doku nutzt nur noch `TrafficExtension`.
* Die Übung im Backlog nutzt noch `WasmPlugin`.

## Konkreter Anwendungsfall: Basic Auth vor einem alten Dienst

**Ausgangslage:** Der Service `reviews` hat keine eigene Anmeldung. Jeder Pod im Mesh, der ihn erreichen darf, bekommt eine Antwort.
Ein Batch-Job eines anderen Teams soll ihn künftig nur noch mit Benutzer und Passwort aufrufen dürfen. Das Team von `reviews` kann den Code gerade nicht ändern.

**Warum hier Wasm:** Istio prüft von Haus aus mTLS-Identitäten und JWT, aber kein Basic Auth. Für genau diesen Fall gibt es ein fertiges Modul aus dem Istio-Ökosystem.

**Lösung:** Der Waypoint prüft den Header `Authorization`, bevor der Request `reviews` erreicht.

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: TrafficExtension
metadata:
  name: basic-auth-for-service
  namespace: bookinfo
spec:
  targetRefs:
    - kind: Service
      group: ""
      name: reviews
  phase: AUTHN
  wasm:
    url: oci://ghcr.io/istio-ecosystem/wasm-extensions/basic_auth:1.12.0
    pluginConfig:
      basic_auth_rules:
        - prefix: "/reviews"
          request_methods:
            - "GET"
            - "POST"
          credentials:
            - "ok:test"
            - "MXQtaW4zOmFkbWluMw=="
```

Was die Felder tun:

* `targetRefs`: gilt nur für den Service `reviews`. Mit `kind: Gateway` und `name: waypoint` gälte es für alle Services hinter dem Waypoint.
* `phase: AUTHN`: der Filter läuft vor den Authentifizierungs-Filtern von Istio. Weitere Werte: `AUTHZ`, `STATS`.
* `wasm.url`: das Modul als OCI-Image. Der Waypoint lädt es selbst aus der Registry.
* `pluginConfig`: Einstellungen, die das Modul liest. Hier: Pfad, Methoden und erlaubte Zugangsdaten (Klartext `user:passwort` oder Base64).

Ergebnis laut Doku:

| Aufruf von `http://reviews:9080/reviews/1` | Antwort |
|---|---|
| ohne Zugangsdaten | `401` |
| mit `Authorization: Basic MXQtaW4zOmFkbWluMw==` | `200` |

Was du dafür brauchst:

* einen Waypoint im Namespace, und `reviews` muss ihn nutzen
* Zugriff vom Waypoint auf die Registry (`ghcr.io` oder eine eigene Registry mit `imagePullSecret`)
* in Produktion: Zugangsdaten nicht im Klartext im Objekt ablegen, und das Modul per `sha256` festnageln

## Referenz

* [Istio: Extensibility (Wasm und Lua)](https://istio.io/v1.31/docs/concepts/extensibility/)
* [Istio: Extend waypoints with WebAssembly plugins](https://istio.io/v1.31/docs/ambient/usage/extend-waypoint-wasm/)
* [Istio: Referenz TrafficExtension](https://istio.io/v1.31/docs/reference/config/proxy_extensions/traffic_extension/)
* [Istio: Referenz WasmPlugin](https://istio.io/v1.31/docs/reference/config/proxy_extensions/wasm-plugin/)
* [Modul basic_auth (Quellcode)](https://github.com/istio-ecosystem/wasm-extensions/tree/master/extensions/basic_auth)
* [Proxy-Wasm Spezifikation](https://github.com/proxy-wasm/spec)
