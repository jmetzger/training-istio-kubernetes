# ArgoCD + Istio Ambient Mode: Zusammenhänge & Praxis

## 1. Architektur-Überblick

### Istio Ambient Mode – Komponenten

Im Gegensatz zum Sidecar-Modus gibt es bei Ambient Mode **kein Envoy-Sidecar** pro Pod.
Stattdessen wird die Mesh-Funktionalität in zwei Schichten aufgeteilt:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Control Plane (istio-system)               │  │
│  │                                                               │  │
│  │   ┌─────────┐     ┌──────────────┐     ┌──────────────────┐  │  │
│  │   │ istiod  │     │  istio-cni   │     │   Gateway API    │  │  │
│  │   │ (xDS)   │     │  (DaemonSet) │     │     CRDs         │  │  │
│  │   └────┬────┘     └──────────────┘     └──────────────────┘  │  │
│  │        │  xDS-Config                                          │  │
│  └────────┼──────────────────────────────────────────────────────┘  │
│           │                                                         │
│  ┌────────┼──────────────────────────────────────────────────────┐  │
│  │  L4    │   Data Plane – ztunnel (DaemonSet, pro Node)        │  │
│  │        ▼                                                      │  │
│  │   ┌─────────┐          ┌─────────┐          ┌─────────┐     │  │
│  │   │ ztunnel │          │ ztunnel │          │ ztunnel │     │  │
│  │   │ Node 1  │◄─HBONE──►│ Node 2  │◄─HBONE──►│ Node 3  │     │  │
│  │   └────┬────┘          └────┬────┘          └────┬────┘     │  │
│  │        │                    │                    │           │  │
│  └────────┼────────────────────┼────────────────────┼───────────┘  │
│           │                    │                    │               │
│  ┌────────┼────────────────────┼────────────────────┼───────────┐  │
│  │  L7    │   Optional: Waypoint Proxies (Envoy, pro Namespace) │  │
│  │        ▼                    ▼                                 │  │
│  │   ┌──────────┐        ┌──────────┐                           │  │
│  │   │ Waypoint │        │ Waypoint │   ← Nur bei L7-Bedarf    │  │
│  │   │ ns: app  │        │ ns: web  │     (AuthzPolicy, Retry) │  │
│  │   └──────────┘        └──────────┘                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Application Pods (KEIN Sidecar!)                            │   │
│  │   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐             │   │
│  │   │App A│  │App B│  │App C│  │App D│  │App E│             │   │
│  │   └─────┘  └─────┘  └─────┘  └─────┘  └─────┘             │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Zusammenfassung der Komponenten:**

| Helm Chart   | Funktion                                  | Typ        |
|-------------|-------------------------------------------|------------|
| `istio/base`  | CRDs (VirtualService, DestinationRule...) | Cluster    |
| `istio/istiod`| Control Plane (xDS, Cert-Mgmt)           | Cluster    |
| `istio/cni`   | CNI Plugin, Traffic-Redirect zu ztunnel  | DaemonSet  |
| `istio/ztunnel`| L4-Proxy, mTLS, HBONE-Tunnel            | DaemonSet  |
| Gateway API   | CRDs für Waypoint/Ingress-Konfiguration  | Cluster    |


---

## 2. Wie ArgoCD mit Istio Ambient zusammenspielt

### GitOps-Workflow

```
┌──────────────┐    Push     ┌──────────────┐    Sync     ┌──────────────┐
│              │────────────►│              │────────────►│              │
│   Developer  │             │   Git Repo   │             │   ArgoCD     │
│              │◄────────────│              │◄────────────│   Controller │
│              │    PR/Review│  (Source of   │   Diff/     │              │
└──────────────┘             │   Truth)     │   Status    └──────┬───────┘
                             └──────────────┘                    │
                                                                 │ Helm
                                                                 │ Template
                                                                 │ + Apply
                                                                 ▼
                             ┌────────────────────────────────────────────┐
                             │          Kubernetes Cluster                │
                             │                                            │
                             │  ┌─────────┐ ┌────────┐ ┌───────────┐    │
                             │  │istio-   │ │istiod  │ │ztunnel    │    │
                             │  │base     │ │        │ │+ istio-cni│    │
                             │  │(CRDs)   │ │(CP)    │ │(DP)       │    │
                             │  └─────────┘ └────────┘ └───────────┘    │
                             │                                            │
                             │  ┌──────────────────────────────────┐     │
                             │  │  App-Namespace (ambient labeled) │     │
                             │  │  ┌──────┐ ┌──────┐ ┌──────────┐ │     │
                             │  │  │App A │ │App B │ │Waypoint  │ │     │
                             │  │  └──────┘ └──────┘ └──────────┘ │     │
                             │  └──────────────────────────────────┘     │
                             └────────────────────────────────────────────┘
```

### Install-Reihenfolge (kritisch!)

ArgoCD deployed Helm Charts. Bei Istio Ambient gibt es **Abhängigkeiten**:

```
  Sync Wave 0        Sync Wave 1       Sync Wave 2        Sync Wave 3
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Gateway API │  │  istio-base  │  │   istiod     │  │  istio-cni   │
│  CRDs        │──►│  (CRDs)      │──►│  (profile:   │──►│  (profile:   │
│              │  │              │  │   ambient)   │  │   ambient)   │
└──────────────┘  └──────────────┘  └──────┬───────┘  └──────┬───────┘
                                           │                  │
                                           │                  ▼
                                           │          ┌──────────────┐
                                           └─────────►│   ztunnel    │
                                                      │              │
                                                      └──────────────┘
                                                              │
                                                              ▼
                                                      ┌──────────────┐
                                                      │  Apps +      │
                                                      │  Namespace   │
                                                      │  Labeling    │
                                                      └──────────────┘
```

**Reihenfolge:** Gateway API CRDs → istio-base → istiod → cni → ztunnel → Apps

---

## 3. Gotchas & Fallstricke

### NetworkPolicy + HBONE Port 15008

Das ist der häufigste Fehler! Ambient Mode nutzt HBONE (HTTP/2 + mTLS) über **Port 15008**.
Bestehende NetworkPolicies blockieren diesen Port oft.

```
  PROBLEM:                                  LÖSUNG:
┌───────────────┐                        ┌───────────────┐
│  NetworkPolicy│                        │  NetworkPolicy│
│               │                        │               │
│  ingress:     │                        │  ingress:     │
│  - port: 8080 │  ← blockiert 15008!   │  - port: 8080 │
│               │                        │  - port: 15008│  ← HBONE erlauben!
└───────────────┘                        └───────────────┘
```

### ArgoCD HA + Ambient Mode

ArgoCD HA nutzt Redis-HA mit Sentinel. Wenn der ArgoCD-Namespace ins Ambient Mesh
eingebunden wird, muss Port 15008 in **allen** ArgoCD-NetworkPolicies erlaubt werden.

**Empfehlung:** ArgoCD-Namespace **NICHT** ins Ambient Mesh einbinden – ArgoCD ist die Management-Ebene für Istio und muss unabhängig davon funktionieren.
ArgoCD verwaltet Istio – es muss nicht selbst im Mesh sein.

### Sidecar-Modus vs. Ambient: GitOps-Vorteil

```
  Sidecar-Modus (imperativ!)          Ambient Mode (deklarativ!)
┌──────────────────────────┐        ┌──────────────────────────┐
│ Sidecar-Version wird bei │        │ ztunnel-Version wird     │
│ Pod-Erstellung bestimmt  │        │ per Helm Chart definiert │
│                          │        │                          │
│ Upgrade = alle Pods      │        │ Upgrade = DaemonSet      │
│ neu starten (imperativ!) │        │ rolling update (deklarativ)│
│                          │        │                          │
│  ✗ Verletzt GitOps-      │        │  ✓ Git = Source of Truth  │
│    Prinzipien            │        │    für Data Plane Version │
└──────────────────────────┘        └──────────────────────────┘
```

---

## 4. Initiales Ausrollen – Schritt für Schritt

```
Was Du tust (manuell)              Was ArgoCD tut (automatisch)
─────────────────────              ────────────────────────────

Schritt 1: Cluster + ArgoCD
  bereitstellen
         │
         ▼
Schritt 2: Git-Repo anlegen
  mit allen Application-YAMLs
         │
         ▼
Schritt 3: root-app.yaml
  kubectl apply (1× manuell!)
         │                         ArgoCD liest apps/ aus Git
         └────────────────────────►      │
                                         ├─► erstellt gateway-api-crds App
                                         ├─► erstellt istio-base App
                                         ├─► erstellt istiod App
                                         ├─► erstellt istio-cni App
                                         ├─► erstellt ztunnel App
                                         └─► erstellt bookinfo App
                                                     │
                                              Sync Waves steuern
                                              die Reihenfolge
                                                     │
                                                     ▼
                                              Istio Ambient läuft!

Ab jetzt: Änderungen NUR noch über Git-Commits.
```

### Schritt 1: Voraussetzungen prüfen

```bash
# Kubernetes >= 1.28
kubectl version

# Helm >= 3.6 (nur zum Prüfen, ArgoCD nutzt Helm intern)
helm version
```

### Schritt 2: ArgoCD installieren

```bash
# Falls noch kein ArgoCD im Cluster:
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Warten bis alles läuft:
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

# Admin-Passwort holen:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Optional: Port-Forward für UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# → https://localhost:8080  (User: admin)
```

### Schritt 3: Git-Repo anlegen und befüllen

```bash
# Neues Repo erstellen (z.B. auf GitHub/GitLab)
mkdir istio-ambient-gitops && cd istio-ambient-gitops
git init

# Verzeichnisstruktur anlegen
mkdir -p apps/istio/values apps/bookinfo
```

Jetzt alle Application-YAMLs und Values-Dateien erstellen
(siehe Abschnitt 4.5 für den Inhalt jeder Datei):

```bash
# Dateien erstellen:
touch apps/istio/gateway-api-crds.yaml
touch apps/istio/istio-base-app.yaml
touch apps/istio/istiod-app.yaml
touch apps/istio/istio-cni-app.yaml
touch apps/istio/ztunnel-app.yaml
touch apps/istio/values/istiod-values.yaml
touch apps/bookinfo/bookinfo-app.yaml

# Alles committen und pushen:
git add -A
git commit -m "Initial Istio Ambient + ArgoCD setup"
git remote add origin https://github.com/DEIN-USER/istio-ambient-gitops.git
git push -u origin main
```

### Schritt 4: root-app.yaml erstellen und anwenden

Diese Datei liegt NICHT im Git-Repo – sie ist der einzige manuelle Schritt:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Schritt 5: Zuschauen und verifizieren

```bash
# ArgoCD Applications beobachten:
kubectl get applications -n argocd -w

# Erwartete Ausgabe nach ein paar Minuten:
# NAME                STATUS   HEALTH
# root                Synced   Healthy
# gateway-api-crds    Synced   Healthy
# istio-base          Synced   Healthy
# istiod              Synced   Healthy
# istio-cni           Synced   Healthy
# ztunnel             Synced   Healthy

# Istio-Pods prüfen:
kubectl get pods -n istio-system

# CRDs prüfen:
kubectl get crds | grep -E 'istio|gateway'
```

### Ab jetzt: Alles über Git

```bash
# Beispiel: Neue App hinzufügen
vim apps/bookinfo/bookinfo-app.yaml
git add -A && git commit -m "Add bookinfo app"
git push
# → ArgoCD erkennt die Änderung und deployed automatisch

# Beispiel: Istio upgraden
# In apps/istio/*-app.yaml: targetRevision von 1.24.2 auf 1.25.0 ändern
git add -A && git commit -m "Upgrade Istio to 1.25.0"
git push
# → ArgoCD rolled die neuen Versionen aus
```

---

### 4.4 Git-Repo Struktur

```
istio-ambient-gitops/
├── apps/
│   ├── istio/
│   │   ├── gateway-api-crds.yaml       # ArgoCD Application
│   │   ├── istio-base-app.yaml         # ArgoCD Application
│   │   ├── istiod-app.yaml             # ArgoCD Application
│   │   ├── istio-cni-app.yaml          # ArgoCD Application
│   │   ├── ztunnel-app.yaml            # ArgoCD Application
│   │   └── values/
│   │       ├── istiod-values.yaml
│   │       ├── cni-values.yaml
│   │       └── ztunnel-values.yaml
│   ├── bookinfo/
│   │   └── bookinfo-app.yaml           # ArgoCD Application
│   └── root-app.yaml                   # App-of-Apps
```

### 4.5 ArgoCD Application Manifeste

#### Gateway API CRDs (Sync Wave -3)

```yaml
# apps/istio/gateway-api-crds.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-api-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/kubernetes-sigs/gateway-api.git
    targetRevision: v1.4.0
    path: config/crd/standard
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=false
```

#### istio-base (Sync Wave -2)

```yaml
# apps/istio/istio-base-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-base
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: base
    targetRevision: 1.24.2     # oder 1.29.x je nach Version
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true    # wichtig für CRDs!
```

#### istiod (Sync Wave -1)

```yaml
# apps/istio/istiod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istiod
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: istiod
      targetRevision: 1.24.2
      helm:
        valueFiles:
          - $values/apps/istio/values/istiod-values.yaml
    - repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
```

**istiod-values.yaml:**
```yaml
# apps/istio/values/istiod-values.yaml
profile: ambient
pilot:
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
meshConfig:
  accessLogFile: /dev/stdout
```

#### istio-cni (Sync Wave 0)

```yaml
# apps/istio/istio-cni-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-cni
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: cni
    targetRevision: 1.24.2
    helm:
      values: |
        profile: ambient
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system       # oder kube-system je nach Setup
  syncPolicy:
    automated:
      selfHeal: true
```

#### ztunnel (Sync Wave 1)

```yaml
# apps/istio/ztunnel-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ztunnel
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: ztunnel
    targetRevision: 1.24.2
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      selfHeal: true
```

#### Multi-Source Variante (alles in einer Application)

Alternativ kann man alle Charts in einer einzigen Application mit `sources` bündeln.
Das funktioniert, aber man verliert die Sync-Wave-Kontrolle:

```yaml
# Kompakte Variante – ACHTUNG: keine Reihenfolge-Garantie!
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-ambient
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  sources:
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: base
      targetRevision: 1.24.2
      helm:
        releaseName: istio-base
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: istiod
      targetRevision: 1.24.2
      helm:
        releaseName: istiod
        parameters:
          - name: profile
            value: ambient
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: cni
      targetRevision: 1.24.2
      helm:
        releaseName: istio-cni
        parameters:
          - name: profile
            value: ambient
    - repoURL: https://istio-release.storage.googleapis.com/charts
      chart: ztunnel
      targetRevision: 1.24.2
      helm:
        releaseName: ztunnel
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### 4.6 App-of-Apps (Root Application)

```yaml
# apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DEIN-USER/istio-ambient-gitops.git
    targetRevision: main
    path: apps/
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
```

### 4.7 Namespace ins Ambient Mesh einbinden

```yaml
# bookinfo-namespace.yaml (im Git-Repo!)
apiVersion: v1
kind: Namespace
metadata:
  name: bookinfo
  labels:
    istio.io/dataplane-mode: ambient    # ← Das ist der Schlüssel!
```

### 4.8 Waypoint Proxy deployen (für L7-Features)

```yaml
# Nur nötig wenn AuthorizationPolicy, Retries, etc. auf L7 gebraucht werden
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-waypoint
  namespace: bookinfo
  labels:
    istio.io/waypoint-for: service      # oder "all" für den ganzen Namespace
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
```

### 4.9 Verifizierung

```bash
# Istio-Komponenten prüfen
kubectl get pods -n istio-system
# Erwartung: istiod-xxx, ztunnel-xxx (pro Node), istio-cni-node-xxx (pro Node)

# CRDs prüfen
kubectl get crds | grep -E 'istio|gateway'

# Ambient-Enrollment prüfen
kubectl get namespace bookinfo --show-labels | grep dataplane-mode

# ztunnel-Logs: mTLS-Verbindungen sichtbar
kubectl logs -n istio-system -l app=ztunnel --tail=20

# Mesh-Status mit istioctl
istioctl ztunnel-config workloads
```

---

## 5. Checkliste: Was muss ich beachten?

```
 ✓  Gateway API CRDs VOR Istio installieren (Sync Wave!)
 ✓  Helm Charts in korrekter Reihenfolge: base → istiod → cni → ztunnel
 ✓  ServerSideApply=true für CRD-haltige Charts setzen
 ✓  profile: ambient bei istiod und cni setzen
 ✓  NetworkPolicies: Port 15008 (HBONE) freigeben
 ✓  ArgoCD-Namespace NICHT ins Ambient Mesh (Management-Ebene bleibt unabhängig)
 ✓  Namespace-Label: istio.io/dataplane-mode=ambient
 ✓  Waypoint nur deployen, wenn L7-Features nötig sind
 ✓  Alle Istio-Versionen einheitlich halten (base, istiod, cni, ztunnel)
 ✓  Bei Upgrades: targetRevision in Git ändern → ArgoCD synced automatisch
```

---

## 6. Upgrade-Workflow

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│ 1. Git Branch  │     │ 2. PR Review   │     │ 3. Merge       │
│                │     │                │     │                │
│ targetRevision │────►│ Diff prüfen:   │────►│ ArgoCD synced  │
│ 1.24.2 → 1.25.0│    │ - Release Notes│     │ automatisch    │
│                │     │ - Breaking Ch. │     │                │
└────────────────┘     └────────────────┘     └────┬───────────┘
                                                    │
                                        ┌───────────┼───────────┐
                                        │           │           │
                                        ▼           ▼           ▼
                                    base CRDs    istiod     ztunnel
                                    (update)    (rollout)  (rollout)
```

**Wichtig:** ztunnel ist ein DaemonSet – beim Upgrade gibt es einen kurzen
Traffic-Umbruch pro Node. In Produktion: `maxUnavailable: 1` in den Values setzen.
