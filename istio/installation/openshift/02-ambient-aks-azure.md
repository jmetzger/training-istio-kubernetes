# Istio Ambient Mode auf Azure (AKS)

Stand: 2026-11-09. Quellen: Microsoft Learn, Solo.io, Istio-Blog (Links unten).
Punkte mit **(ungeprueft)** stammen aus Community-Beiträgen oder Erfahrungswissen und sind noch
nicht gegen die offizielle Doku der eingesetzten Versionen verifiziert.

Hier geht es um **AKS** (Azure Kubernetes Service), nicht um OpenShift. Für OpenShift siehe
[Istio Ambient auf OpenShift (OSSM 3)](01-ossm-ambient-ocp.md).

## Kurzfassung

- Das **Istio-Add-on von AKS** (von Microsoft verwaltet) unterstützt **Ambient noch nicht**.
  Laut Microsoft steht die Integration auf der Roadmap.
- Ambient läuft auf AKS, wenn du **Upstream-Istio selbst installierst** (`istioctl` oder Helm,
  Profil `ambient`). Das ist derselbe Weg wie in diesem Training.
- Microsoft hat zusätzlich **Azure Kubernetes Application Network** (Preview). Es nutzt Ambient und
  die Gateway API als verwaltetes Angebot.
- Das GA-Istio-Add-on bleibt laut Microsoft der empfohlene Einstieg. Es nutzt Sidecars.

## Die Optionen im Vergleich

| Option | Ambient | Verwaltet von | Hinweis |
|--------|---------|---------------|---------|
| AKS Istio-Add-on | nein (Roadmap) | Microsoft | Sidecar-Modus. Gateway API für Ingress und GAMMA laut Doku noch nicht unterstützt |
| Azure Kubernetes Application Network | ja (Preview) | Microsoft | Preview, nicht für Produktion gedacht |
| Upstream-Istio per `istioctl` oder Helm | ja | ihr selbst | volle Kontrolle, kein Microsoft-Support für das Mesh |
| Solo.io AKS-Extension | ja | Solo.io (kommerziell) | Drittanbieter, eigener Support |

## Voraussetzungen und Stolpersteine bei selbst installiertem Ambient

- **Netzwerk-Plugin:** Azure CNI funktioniert. Mit **kubenet** läuft Ambient nach einer älteren
  Quelle nicht. Azure CNI Overlay: keine belastbare aktuelle Aussage gefunden **(ungeprueft)**.
- **istio-cni** läuft als verketteter Plugin neben Azure CNI und ersetzt es nicht.
- **Port 15008 (HBONE)** muss zwischen den Nodes durch NSG und Firewall erlaubt sein.
- **Cilium** (Azure CNI powered by Cilium): Community-Beiträge nennen für Cilium ohne kube-proxy
  `socketLB.hostNamespaceOnly: true` und `cni.exclusive: false`. Gegen die Doku von Cilium und
  Istio prüfen **(ungeprueft)**.
- **Cilium-mTLS von Azure (Preview, März 2026)** überschneidet sich mit der Aufgabe von ztunnel.
  Beides gleichzeitig für dieselben Workloads wäre doppelt. Eins von beiden wählen.
- **Add-on und Eigeninstallation nicht mischen:** Läuft das AKS-Istio-Add-on im Cluster, gibt es
  Konflikte mit einer selbst installierten Istio-Version. Vorher prüfen, ob das Add-on deaktiviert
  ist **(ungeprueft)**.
- **Gateway API CRDs** vor der Installation sicherstellen. Ob AKS sie mitbringt, vorab prüfen
  **(ungeprueft)**.

## Installation (wie im Training)

Dieselben Schritte wie auf dem kubeadm-Cluster, siehe
[istio-installation mit istioctl - ambient](../ambient/03-install-with-istioctl-ambient.md):

```
istioctl install --set profile=ambient
```

Danach Namespace mit `istio.io/dataplane-mode=ambient` labeln und bei Bedarf einen Waypoint anlegen.

## Empfehlung

- Wenn Ambient das Ziel ist: **Upstream-Istio selbst installieren** (`istioctl` oder Helm) und
  Azure CNI nutzen.
- Das verwaltete Add-on nur nehmen, wenn Sidecar ausreicht und Microsoft-Support wichtiger ist.
- Application Network als Preview beobachten.
- Vor dem Einsatz in der Support-Matrix von Microsoft die aktuelle Aussage zu Ambient prüfen,
  weil sich der Stand schnell ändert.

## Quellen

- [Microsoft Learn: Istio-based service mesh add-on for AKS](https://learn.microsoft.com/azure/aks/istio-about)
- [Microsoft Learn: Application Network, Traffic-Management-Anwendungsfälle](https://learn.microsoft.com/azure/application-network/traffic-management-use-cases)
- [Solo.io: Istio Ambient Mesh auf AKS](https://www.solo.io/blog/istio-ambient-mesh-azure-kubernetes-service)
- [Solo.io: AKS-Extension für Ambient](https://docs.solo.io/istio/1.30.x/ambient/setup/install/aks_extension/)
- [Istio-Blog: Ambient mit verschiedenen Kubernetes-Anbietern und CNIs](https://Istio.io/blog/2024/inpod-traffic-redirection-ambient/)
- [Microsoft Tech Community: Cilium mTLS für Azure (Preview)](https://techcommunity.microsoft.com/t5/azure-networking-blog/announcing-public-preview-cilium-mtls-encryption-for-azure/ba-p/4504423)
