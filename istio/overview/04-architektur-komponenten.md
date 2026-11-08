# Architektur & Komponenten von Istio (Ambient Mode)

Im Ambient Mode steckt kein Envoy-Sidecar mehr in jedem Pod. Die Aufgaben sind stattdessen auf **zwei Ebenen** verteilt:

| Ebene | Was | Wo |
|---|---|---|
| **Layer 4 (immer)** | mTLS, Identität, L4-Policies, TCP-Metriken | `ztunnel`, ein Agent pro Node |
| **Layer 7 (nur bei Bedarf)** | Routing, Retries, Timeouts, L7-Policies, HTTP-Metriken | `Waypoint`, ein Envoy pro Namespace |

Dazu kommen der `istiod` als Control Plane und `istio-cni`, der den Traffic der Pods zum ztunnel umleitet.

## Überblick

![Istio Ambient: Komponenten](../../images/istio/ambient-komponenten.svg)

## Die Komponenten im Einzelnen

| Komponente | Läuft als | Aufgabe | Zuständig (bei Plattform-Betrieb) |
|---|---|---|---|
| **istiod** | Deployment, `istio-system` | Verteilt Konfiguration per xDS, stellt Zertifikate aus (CA), Service Discovery | Plattform |
| **istio-cni** | DaemonSet, ein Pod pro Node | Richtet die Umleitung des Pod-Traffics zum ztunnel ein. Dafür sind keine `NET_ADMIN`-Rechte in den Anwendungs-Pods nötig | Plattform |
| **ztunnel** | DaemonSet, ein Pod pro Node | Layer 4: baut mTLS-Tunnel (HBONE) auf, prüft L4-Policies, liefert TCP-Metriken. Hat pro Pod auf dem Node eine eigene Identität (SPIFFE) | Plattform |
| **Waypoint** | Deployment, per Gateway API (`gatewayClassName: istio-waypoint`) im Namespace | Layer 7: HTTPRoute, Retries, Timeouts, Fault Injection, L7-`AuthorizationPolicy`, HTTP-Metriken | **Namespace-Team** |
| **Ingress Gateway** | Deployment, per Gateway API (`gatewayClassName: istio`) | Eingang von außen in das Mesh | Namespace-Team oder Plattform |

Die Spalte ganz rechts ist für das Training wichtig ("Istio als Mieter"): ztunnel, istio-cni, istiod und die CRDs gehören zur Plattform.
Im eigenen Namespace sind nur das **Label** (`istio.io/dataplane-mode=ambient`), der **Waypoint**, die **Routen** und die **Policies** Sache des Teams.

### HBONE in zwei Sätzen

HBONE (HTTP-Based Overlay Network Environment) ist der Tunnel zwischen den ztunnels: HTTP/2 `CONNECT` über mTLS auf **Port 15008**.
Mehrere Verbindungen zwischen zwei Nodes laufen gemultiplext durch denselben Tunnel, und die Identität des Absenders (SPIFFE) reist im Zertifikat mit.

## Datenpfad: ohne und mit Waypoint

![Istio Ambient: Datenpfad](../../images/istio/ambient-datenpfad.svg)

* **Ohne Waypoint** (Secure Overlay): zwei Hops, nur Layer 4. Reicht für mTLS und einfache Policies (wer darf mit wem sprechen).
* **Mit Waypoint:** ein zusätzlicher Hop für alles, was HTTP-Wissen braucht. Man aktiviert ihn gezielt pro Namespace oder Service (`istio.io/use-waypoint`).
* Beides kann im selben Cluster nebeneinander laufen, Namespace für Namespace.

## Ablauf: Konfiguration

![Istio Ambient: Ablauf Konfiguration](../../images/istio/ambient-ablauf-konfiguration.svg)

1. Der Admin legt eine Route oder Policy an (`kubectl apply`).
2. istiod validiert sie und übersetzt sie in Proxy-Konfiguration.
3. istiod schickt die Konfiguration per xDS an ztunnel und Waypoint.

Die ztunnels holen dabei auch die Zertifikate für die Pods auf ihrem Node, der Waypoint holt seines bei istiod (CA).
Änderungen wirken ohne Pod-Neustart.

## Ablauf: Request

![Istio Ambient: Ablauf Request](../../images/istio/ambient-ablauf-request.svg)

1. Ein Pod ruft einen Service auf. `istio-cni` hat den Traffic zum lokalen ztunnel umgeleitet.
2. Der ztunnel schickt ihn per HBONE (mTLS) weiter, mit Waypoint zuerst dorthin.
3. Der Waypoint wendet die Layer-7-Regeln an (Routing, Retries, L7-Policies).
4. Weiter geht es per HBONE zum ztunnel auf dem Ziel-Node.
5. Der ztunnel prüft die L4-Policy.
6. Er übergibt den Request an den Ziel-Pod.

Ohne Waypoint entfallen die Schritte 3 und 4: Der ztunnel auf Node 1 spricht direkt mit dem ztunnel auf Node 2.

## Ambient im Vergleich zu Sidecar

| | **Sidecar** | **Ambient** |
|---|---|---|
| Wo läuft der Proxy? | Envoy in **jedem Pod** | ztunnel **pro Node**, Waypoint **pro Namespace** (optional) |
| Pod-Neustart zum Aktivieren? | Ja, der Sidecar wird injiziert | Nein, Label setzen genügt |
| Ressourcen | CPU/RAM pro Pod | deutlich weniger Overhead |
| Layer 7 | immer (Envoy im Pod) | nur mit Waypoint |
| mTLS | Sidecar zu Sidecar | ztunnel zu ztunnel (HBONE) |
| Upgrade von Istio | Pods müssen neu starten | ztunnel/CNI als DaemonSet, Anwendungs-Pods bleiben unberührt |
| Koexistenz | ja, beide Modi können im selben Cluster laufen | |

Die klassische Sidecar-Architektur steht in [overview-classic-sidecar.md](overview-classic-sidecar.md) und
[07-istio-proxy-concepts-sidecar.md](07-istio-proxy-concepts-sidecar.md).

## Selbst nachschauen (auf dem eigenen Cluster)

```bash
# Control Plane und Node-Agents: je 1 ztunnel und 1 istio-cni pro Node, 1 istiod
kubectl get deploy,ds -n istio-system
kubectl get pods -n istio-system -o wide

# Die Anwendungs-Pods haben keinen Sidecar (1/1)
kubectl get pods -n bookinfo

# Ist der Namespace im Mesh, und nutzt er einen Waypoint?
kubectl get ns bookinfo --show-labels
kubectl get gateway -n bookinfo

# Was weiß ztunnel? Welcher Pod spricht HBONE, welcher hat einen Waypoint?
istioctl ztunnel-config workload
```

## Merksätze

* **ztunnel = Layer 4, pro Node, immer da.** Er verschlüsselt und prüft, wer mit wem sprechen darf.
* **Waypoint = Layer 7, pro Namespace, nur wenn man ihn braucht.** Er routet und wiederholt Requests.
* **istiod verteilt nur Konfiguration und Zertifikate**, der Traffic fließt nie durch ihn.
* Ambient verändert die Pods nicht, deshalb ist der Einstieg ein Label und der Ausstieg ebenfalls.
