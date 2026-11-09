# Istio Ambient Mode auf OpenShift (OCP, OSSM 3)

Stand: 2026-11-09. Quelle: Red-Hat-Doku und Blog zu OSSM 3.0 bis 3.3 (Links unten).
Punkte mit **(ungeprueft)** stammen aus Erfahrungswissen und sind noch nicht gegen die
englische Doku verifiziert. Für Azure siehe
[Istio Ambient auf Azure (AKS)](02-ambient-aks-azure.md).

## Kurzfassung

- Ambient laeuft auf OpenShift ueber **Red Hat OpenShift Service Mesh 3 (OSSM 3)**.
- OSSM 3 basiert auf dem **Sail Operator** und wird ueber **OperatorHub/OLM** installiert,
  **nicht** ueber Helm-Charts.
- Produktiv sinnvoll ab **OSSM 3.2** (Ambient GA, braucht OCP 4.18 oder neuer).
- Pro Cluster gibt es **ein** Ambient-Mesh. Mischbetrieb mit Sidecars ist nur Preview.

## Support-Stand pro Version

| OSSM | Istio | Ambient-Status |
|------|-------|----------------|
| 3.0  | 1.24  | Developer Preview |
| 3.1  | 1.26  | Technology Preview, nur auf Clustern ohne bestehende OSSM-Installation |
| 3.2  | 1.27  | **GA** fuer ztunnel und waypoint. Bekannt: nicht auf FIPS-Clustern |
| 3.3  | -     | FIPS-Support fuer ztunnel. Multi-Cluster-Ambient als Tech Preview |

Upgrade von OSSM 2.x: erst auf 3.0, dann 3.1, dann 3.2.

## Installation

Zwei Stufen.

### 1. Operator per OLM installieren

Web-Konsole: Operators, OperatorHub, "Red Hat OpenShift Service Mesh 3", Install.
Oder als YAML (Paketname und Channel vorher mit `oc get packagemanifest` pruefen):

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator3
  namespace: openshift-operators
spec:
  channel: stable
  name: servicemeshoperator3
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### 2. Mesh ueber Custom Resources anlegen

Die Namespaces `istio-system`, `istio-cni` und `ztunnel` vorher anlegen.
`apiVersion` und Felder gegen die CRDs der installierten Version pruefen
(`oc explain istio.spec`).

```yaml
apiVersion: sailoperator.io/v1
kind: Istio
metadata: { name: default }
spec:
  namespace: istio-system
  profile: ambient
  values:
    pilot:
      trustedZtunnelNamespace: ztunnel
---
apiVersion: sailoperator.io/v1
kind: IstioCNI
metadata: { name: default }
spec:
  namespace: istio-cni
  profile: ambient
---
apiVersion: sailoperator.io/v1
kind: ZTunnel
metadata: { name: default }
spec:
  namespace: ztunnel
```

Namespace ins Mesh aufnehmen: `oc label namespace <ns> istio.io/dataplane-mode=ambient`.
L7-Funktionen ueber einen Waypoint (Gateway API, `gatewayClassName: istio-waypoint`).

### Was OperatorHub ist

- OLM (Operator Lifecycle Manager) ist Teil von OpenShift und laeuft im Cluster.
- Kataloge sind `CatalogSource`-Objekte in `openshift-marketplace`
  (`redhat-operators`, `certified-operators`, `community-operators`, `redhat-marketplace`).
- Die Web-Konsole (Operators, OperatorHub) ist nur eine Oberflaeche. Dahinter entsteht eine
  `Subscription`. Dasselbe geht per `oc apply` und damit per GitOps.
- Zum Installieren braucht man cluster-admin oder entsprechende Rechte.

## Unterschiede: OSSM (Operator) gegen Upstream-Istio (Helm)

| Thema | OSSM 3 (Operator) | Upstream-Istio per Helm |
|-------|-------------------|-------------------------|
| Support | Red Hat | keiner von Red Hat fuer den Mesh-Teil |
| Ausrollen | Subscription plus CRs (`Istio`, `IstioCNI`, `ZTunnel`) | `helm install` (base, istiod, cni, ztunnel) |
| Sichtbar als | Operator, keine Helm-Releases | Helm-Releases |
| Konfiguration | `spec.values` (entspricht den Istio-Helm-Values) | `values.yaml` |
| SCCs, privilegierte Pods | vom Operator behandelt | selbst loesen |
| Upgrades | Revisionen und Canary ueber `IstioRevision`, `IstioRevisionTag` | selbst steuern |
| Versionen | an OSSM-Release gebunden (z. B. 3.2 gleich Istio 1.27) | frei waehlbar, immer aktuell |
| Gateway-API-CRDs | auf OCP vom Ingress Operator verwaltet, OSSM darauf abgestimmt | selbst abstimmen, nicht ueberschreiben |
| Kiali, Monitoring | integriert und getestet | selbst integrieren |
| Kataloge | braucht OLM und Katalog (bei Air-Gap gespiegelt) | braucht nur eine Chart-Quelle |

Intern nutzt der Sail Operator die Istio-Helm-Charts zum Rendern. Du siehst sie aber nicht.

Upstream bringt eigene OpenShift-Profile mit (`openshift`, `openshift-ambient`) **(ungeprueft)**.

## Einschraenkungen von Ambient in OSSM (Stand 3.3)

| Thema | Status |
|-------|--------|
| ztunnel (Kern) | GA |
| Waypoint (Kern) | GA |
| Waypoint mit Gateway API Stable Channel (HTTPRoute, GRPCRoute) | GA |
| Sidecar- und Ambient-Namespaces im selben Mesh mischen | Developer Preview |
| Dual-Stack und IPv6-only | Technology Preview |
| Multi-Cluster, Multi-Primary | Technology Preview |
| Andere Multi-Cluster-Topologien | nicht verfuegbar |
| Ambient auf Cluster mit bestehendem Sidecar-Mesh | nicht verfuegbar |
| Mehrere Ambient-Meshes in einem Cluster | nicht verfuegbar |
| FIPS-Cluster | nicht in 3.2, ab 3.3 unterstuetzt |
| Post-Quantum-Kryptografie (X25519MLKEM768) auf FIPS-Clustern | nicht unterstuetzt |

Wichtig fuer IPv6-Umgebungen: IPv6-only ist bei Ambient in OSSM nur Tech Preview.

## Weitere Punkte beim Betrieb

- **L4 ohne Waypoint:** mTLS und `AuthorizationPolicy` auf Identitaet. L7-Policies brauchen
  einen Waypoint pro Namespace oder Service.
- **NetworkPolicies und AdminNetworkPolicies:** HBONE-Port 15008 und Health-Probes muessen
  durchgelassen werden **(ungeprueft)**.
- **CNI:** OVN-Kubernetes ist der Normalfall. ztunnel und CNI-Chaining dort vor dem Training testen.
- **Ingress:** Istio-Gateway kann parallel zum OpenShift-Router (HAProxy) laufen.
- **Observability:** ueber Kiali und den OpenShift-Monitoring-Stack (User-Workload-Monitoring).
  Eigener Konfigurationsblock.
- **Air-Gap:** ohne Internet braucht der Cluster einen gespiegelten Operator-Katalog
  (`oc-mirror`, eigene `CatalogSource`).
- **Rechte:** Operator-Installation nur mit cluster-admin.
- **OKD:** OperatorHub gibt es, aber mit `community-operators`. Der OSSM-Operator liegt in
  `redhat-operators` und setzt eine OCP-Subscription voraus.
- **GitOps:** Reihenfolge einhalten. Subscription zuerst, warten bis die CRDs da sind, dann die
  CRs (z. B. Sync-Waves in Argo CD).

## Empfehlung

- OSSM 3.2 oder 3.3 per Operator als Standard. 3.3, falls FIPS gebraucht wird.
- Ein Ambient-Mesh pro Cluster, kein Mischbetrieb.
- Vor der Planung klaeren: connected oder disconnected, wer darf Operatoren installieren,
  braucht ihr Multi-Cluster oder IPv6-only (dann nur Tech Preview).
- Helm/Upstream nur, wenn es keine Red-Hat-Subscription gibt (OKD) oder ein Feature noetig ist,
  das OSSM nicht liefert.

## Quellen

- [OSSM 3.3 Feature-Support-Tabellen](https://docs.redhat.com/it/documentation/red_hat_openshift_service_mesh/3.3/html/release_notes/ossm-release-notes-feature-support-tables)
- [OSSM 3.2 Release Notes](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.2/html/release_notes/ossm-release-notes)
- [OSSM 3.3 Release Notes](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.3/html/release_notes/ossm-release-notes)
- [Blog: OSSM 3.2 und Istio Ambient Mode](https://www.redhat.com/en/blog/introducing-openshift-service-mesh-32-istios-ambient-mode)
- [Blog: OSSM 3.1](https://www.redhat.com/de/blog/introducing-openshift-service-mesh-31)
- [Blog: OSSM 3.3 und Post-Quantum-Kryptografie](https://www.redhat.com/es/blog/openshift-service-mesh-33-adds-post-quantum-cryptography)
