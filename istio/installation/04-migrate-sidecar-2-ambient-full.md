# Migrate from sidecar-2-ambient (full walktrough from scratch)

  * This is a full walkthrough without having istio at all  
  * (von Bookinfo-Sidecar-Setup zu Ambient mit Waypoint & AuthorizationPolicies)
---

## A. Zielbild & Voraussetzungen

### A1. Zielbild

- **Startzustand**:
  - Istio im **Sidecar-Modus** installiert (demo-Setup).
  - Bookinfo-App läuft, `productpage` von außen über Gateway erreichbar.
  - Bestehende **AuthorizationPolicies mit `selector`** (Sidecar-Stil) regeln:
    - `productpage → details` (GET)
    - `reviews → ratings` (GET/POST)
    - `gateway → productpage` (L4, kein Waypoint nötig)

- **Endzustand**:
  - Istio läuft im **Ambient-Mode** (Profil `ambient`).
  - Namespace `bookinfo` ist mit `istio.io/dataplane-mode=ambient` gelabelt.
  - **Keine Sidecars** mehr, nur ztunnel + Waypoint.
  - L7-Policies werden über **Waypoint + `targetRefs`** durchgesetzt.

---

### A2. Technische Voraussetzungen

- Laufender Kubernetes-Cluster (kind, k3d, Minikube, EKS, GKE, …).
- `kubectl` konfiguriert.
- `curl`, `bash`.
- Internetzugang aus der Shell, um Istio & CRDs zu holen.

```bash
kubectl cluster-info
kubectl get nodes
