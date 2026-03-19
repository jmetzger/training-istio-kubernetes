# ArgoCD – Überblick und Nutzen für Istio

## Was ist ArgoCD?

ArgoCD ist ein deklarativer GitOps-Controller für Kubernetes. Er überwacht Git-Repositories und synchronisiert den dort definierten Soll-Zustand automatisch mit dem Cluster. Abweichungen (Drift) werden erkannt, gemeldet und optional selbstständig korrigiert.

Kernprinzip: **Git ist die Single Source of Truth** – kein `kubectl apply` oder `helm install` mehr von Hand.

## Architektur-Übersicht

![ArgoCD + Istio GitOps Architektur](argocd-istio-overview.svg)

## Warum ArgoCD für Istio?

Istio bringt eine Vielzahl an Custom Resources mit (VirtualService, DestinationRule, AuthorizationPolicy, Gateway, PeerAuthentication, EnvoyFilter, WasmPlugin …). Diese manuell über mehrere Cluster oder Namespaces konsistent zu halten, ist fehleranfällig. ArgoCD löst genau dieses Problem.

### Konkrete Einsatzszenarien

| Szenario | Ohne ArgoCD | Mit ArgoCD |
|---|---|---|
| **Istio-Installation & Upgrades** | `istioctl install` oder Helm manuell ausführen, Version im Cluster unklar | Istio Helm Charts im Git versioniert, Upgrade = Commit + PR |
| **Traffic Management** | VirtualServices per `kubectl apply` deployen, kein Review-Prozess | Canary Releases, Traffic Shifting als PR reviewbar und auditierbar |
| **Security Policies** | AuthorizationPolicy/PeerAuthentication ad hoc anwenden, Drift möglich | Zero-Trust-Policies versioniert, Drift wird erkannt und korrigiert |
| **Multi-Cluster / Multi-Env** | Copy-Paste von Manifests zwischen Staging und Prod | Kustomize-Overlays pro Umgebung, ein Git-Repo als Quelle |
| **Observability-Stack** | Kiali, Prometheus, Grafana manuell installieren | Gesamter Observability-Stack deklarativ verwaltet |

## Vorteile im Überblick

- **Auditierbarkeit** – Jede Änderung an Istio-Konfiguration ist ein Git-Commit mit Autor, Timestamp und Diff.
- **Rollback in Sekunden** – Git revert → ArgoCD synchronisiert den vorherigen Zustand.
- **Drift Detection** – Manuelle Änderungen im Cluster (`kubectl edit`) werden erkannt und gemeldet.
- **Self-Healing** – Optional: ArgoCD stellt den Git-Zustand automatisch wieder her.
- **PR-basierte Workflows** – Istio-Konfigurationsänderungen durchlaufen Code Review, bevor sie live gehen.
- **Multi-Cluster-Fähigkeit** – Ein ArgoCD verwaltet Istio-Konfiguration über mehrere Cluster hinweg.
- **Helm & Kustomize nativ** – Istio Helm Charts und Kustomize-Overlays werden direkt unterstützt.

## ArgoCD vs. Flux CD

| Kriterium | ArgoCD | Flux CD |
|---|---|---|
| UI | Vollständiges Web-UI mit Ressourcen-Baum | Kein eigenes UI (Weave GitOps als Add-on) |
| Architektur | Zentraler Server + Application CRD | Dezentrale Controller (source, kustomize, helm) |
| Multi-Tenancy | App Projects mit RBAC | Namespace-basierte Isolation |
| Sync-Modell | Pull + manueller/automatischer Sync | Reiner Pull-basierter Reconciliation-Loop |
| Istio-Kompatibilität | Volle CRD-Unterstützung, Resource Hooks für Reihenfolge | Volle CRD-Unterstützung, Depends-on für Reihenfolge |

Beide Tools sind für Istio gleich gut geeignet. ArgoCD punktet durch das UI (hilfreich für Trainings und Troubleshooting), Flux CD durch geringere Komplexität und bessere Composability.

## Typische Repo-Struktur für Istio + ArgoCD

```
gitops-repo/
├── base/
│   ├── istio/
│   │   ├── istio-base/          # Helm values für istio/base
│   │   ├── istiod/              # Helm values für istiod
│   │   └── gateway/             # Helm values für Istio Gateway
│   ├── bookinfo/
│   │   ├── deployment.yaml
│   │   ├── virtualservice.yaml
│   │   └── destinationrule.yaml
│   └── policies/
│       ├── authz-deny-all.yaml
│       └── peer-authn-strict.yaml
├── overlays/
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
└── argocd/
    ├── app-istio.yaml           # ArgoCD Application für Istio
    ├── app-bookinfo.yaml
    └── app-policies.yaml
```

## Fazit

ArgoCD ist kein Muss für Istio, aber ein starker Enabler: Es macht Istio-Konfiguration nachvollziehbar, reviewbar und reproduzierbar. Gerade bei komplexeren Setups (Multi-Cluster, viele Policies, häufige Traffic-Shifts) reduziert es Fehler und erhöht die Geschwindigkeit.
