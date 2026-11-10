# Was Ambient (noch) nicht kann, was Sidecar kann

**Stand: 10.11.2026.** Grundlage ist die Istio-Doku zu Version 1.32. Im Training läuft **1.31.4**; die Aussagen gelten vermutlich auch dafür, abgeglichen ist das nicht.

## Kurzfassung

Für **einen einzelnen Kubernetes-Cluster** mit Gateway API, ztunnel und Waypoint (unser Aufbau) fehlt fast nichts.
Die Lücken liegen bei VMs, EnvoyFilter, SPIRE und Multi-Cluster.

## Was es nur mit Sidecar gibt

| Thema | Sidecar | Ambient |
|---|---|---|
| VMs im Mesh | geht | nicht möglich |
| EnvoyFilter | geht | nicht unterstützt, auch nicht am Waypoint |
| SPIRE-Integration | geht | nicht unterstützt |
| PeerAuthentication `DISABLE` | geht | wird ignoriert |
| Multi-Cluster | stabil | nur Multi-Primary (Primary-Remote geht nicht) |
| Multi-Network | stabil | laut Doku Beta |
| Sidecar-Client → Waypoint | geht | Waypoint wird umgangen (siehe unten) |

## Was Ambient nur eingeschränkt kann

* **Layer 7 braucht einen Waypoint.** Nur mit ztunnel gibt es mTLS, L4-Autorisierung und TCP-Metriken, aber kein HTTP-Routing, kein HTTP-Tracing.
* **`VirtualService`** ist in Ambient noch Alpha. Besser `HTTPRoute` (Gateway API) nehmen, und beides nicht für denselben Workload mischen.
* **Ingress-Gateways (istio-ingressgateway)** nutzen den Waypoint standardmäßig nicht. Mit dem Label `istio.io/ingress-use-waypoint=true` am Service oder Namespace (Namespace ab Istio 1.25) geht es doch.
  Laut Istio-Doku gilt das nur, wenn istiod die Variable `ENABLE_INGRESS_WAYPOINT_ROUTING` aktiv hat (Standard `false`).
  **Auf unserem Cluster mit 1.31.4 ist die Variable nicht gesetzt, das Label wirkte trotzdem** (getestet in [Übung Zero Trust Ambient](../exercises/19-zerotrust-bookinfo-ambient.md)).
  Warum, ist nicht geklärt. Wirkt das Label bei dir nicht, zuerst die Variable an istiod prüfen.
* **Sicherheitsmodell:** Ambient "stark", Sidecar "am stärksten". Der ztunnel hält pro Node die Schlüssel aller dortigen Workloads, der Sidecar nur die seines Pods.

## Ersatz für EnvoyFilter: `TrafficExtension` (Alpha)

Einen 1:1-Ersatz gibt es nicht. Erweiterungen laufen nur **am Waypoint**, mit Wasm oder Lua (beides Alpha).

```yaml
apiVersion: extensions.istio.io/v1alpha1
kind: TrafficExtension
metadata:
  name: basic-auth-at-waypoint
spec:
  targetRefs:
    - kind: Gateway
      group: gateway.networking.k8s.io
      name: waypoint
  phase: AUTHN
  wasm:
    url: oci://ghcr.io/istio-ecosystem/wasm-extensions/basic_auth:1.12.0
```

* `targetRefs` zeigt auf den Waypoint (oder einen Service) und ersetzt den `selector` aus dem Sidecar-Modus.
* Für Lua steht das Skript unter `lua.inlineCode` statt `wasm`.
* Es sind nur HTTP-Filter. Cluster, Listener oder Filterketten patchen wie mit `EnvoyFilter` geht nicht.
* Im ztunnel gibt es keine Erweiterungen.

Ob in 1.31 noch `WasmPlugin` der richtige Weg ist, ist nicht geprüft.

## Mischbetrieb: Sidecar-Client → Ambient-Ziel

Sidecar und Ambient können im selben Mesh laufen. Die Falle (aus der Doku):

> Traffic from sidecar mode workloads bypasses waypoint proxies.

![Sidecar-Client umgeht den Waypoint](../../images/istio/sidecar-zu-ambient-waypoint-umgehung.svg)

Es gibt keinen Fehler, die Anfrage kommt an. Aber **L7-Policies, HTTPRoute-Routing und L7-Telemetrie des Waypoints greifen nicht.**

| Pfad | L7 |
|---|---|
| Sidecar → Sidecar | wie gewohnt |
| Ambient → Ambient mit Waypoint | greift am Waypoint |
| Ambient → Sidecar | vermutlich am Ziel-Sidecar (nicht getestet) |
| **Sidecar → Ambient mit Waypoint** | **Waypoint wird umgangen** |

### Folgen für die Migration

* **Nur mTLS und L4-Policies:** kein Problem, das macht der ztunnel allein.
* **Mit L7-Policies:** Ziel und alle seine Clients **zusammen** umstellen, nicht einzelne Services quer durch den Aufrufgraphen.
* **Zero-Downtime geht mit L7-Policies nicht.** Die alten selektorbasierten Policies müssen weg, bevor die waypoint-basierten greifen. Dazwischen fehlt die L7-Durchsetzung, also Wartungsfenster einplanen.
* **Vorher prüfen, wer den Service aufruft** (Kiali, Metriken, auch Jobs, Cronjobs, andere Namespaces).
* **Waypoint erzwingen:** Eine `AuthorizationPolicy`, die nur die Waypoint-Identität zulässt, lässt umgangenen Verkehr sichtbar scheitern. In der Mischphase bricht das alle Sidecar-Clients ab, also bewusst einsetzen.
* **Dauerzustand ist das fragil:** Ein neuer Client mit Sidecar, und die L7-Regel wirkt nicht mehr. Mischbetrieb nur als Übergang.

Reihenfolge laut Doku: Ambient installieren → Policies migrieren (bei reinem L4 entfällt das) → Namespace für Ambient labeln → Waypoints aktivieren → Sidecar-Injection entfernen → Pods neu starten. Jeder Schritt ist einzeln rückgängig zu machen.

## Chaos-Tests / Fault Injection (Delay, Abort)

**Nicht verifiziert.** In der offiziellen Doku gibt es dazu keine bestätigte Aussage für Ambient.

* Die Aufgabe "Fault Injection" ([istio.io](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)) nutzt `VirtualService` (`fault.delay`, `fault.abort`) und sagt nichts zu Ambient oder Waypoint.
* `HTTPRoute` hat kein Feld für Fault Injection. `VirtualService` ist in Ambient Alpha, und Mischen mit Gateway API "will lead to undefined behavior".
* Auf L4 (nur ztunnel) gibt es keine Fault Injection.
* Blogs (keine offizielle Quelle) nennen: `VirtualService` am Waypoint.

**Fürs Training:** Der Chaos-Link in der Agenda führt in die Sidecar-Welt. Vor der Demo auf dem Ambient-Cluster ausprobieren, ob ein `VirtualService` mit `fault` am Waypoint wirkt.

## Gateway API wird intern zu `VirtualService`

istiod übersetzt `HTTPRoute`, `GRPCRoute`, `TCPRoute` und `TLSRoute` intern in `VirtualService`-Objekte
(`pilot/pkg/config/kube/gateway/route_collections.go`, Branch `master`).
Die gibt es nur im Speicher von istiod, `kubectl get virtualservice` zeigt sie nicht. Für Waypoints ist das nicht gesondert geprüft.

## Wann bleibt man bei Sidecar?

Wenn eines davon zutrifft: **VMs im Mesh, `EnvoyFilter`, SPIRE, Multi-Cluster mit Primary-Remote.**
Das sind laut Doku die echten Gründe gegen Ambient.

## Quellen

* [Istio: Sidecar or ambient?](https://istio.io/latest/docs/overview/dataplane-modes/)
* [Istio: Migrate from Sidecar to Ambient](https://istio.io/latest/docs/ambient/migrate/)
* [Istio: Waypoint-Nutzung](https://istio.io/latest/docs/ambient/usage/waypoint/)
* [Istio: Extend waypoints with Wasm](https://istio.io/latest/docs/ambient/usage/extend-waypoint-wasm/)
* [Istio: Extend waypoints with Lua](https://istio.io/latest/docs/ambient/usage/extend-waypoint-lua/)
* [Istio 1.32 Ankündigung](https://istio.io/latest/news/releases/1.32.x/announcing-1.32/)
