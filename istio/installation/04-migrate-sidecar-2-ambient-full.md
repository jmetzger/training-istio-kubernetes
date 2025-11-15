# Migrate from sidecar-2-ambient (full walktrough from scratch)

  * This is a full walkthrough without having istio at all  
  * (von Bookinfo-Sidecar-Setup zu Ambient mit Waypoint & AuthorizationPolicies)

## A. Zielbild & Voraussetzungen

### A1. Zielbild

* **Startzustand**:

  * Istio im **Sidecar-Modus** installiert (demo-Setup).
  * Bookinfo-App läuft, `productpage` von außen über Gateway erreichbar.
  * Bestehende **AuthorizationPolicies mit `selector`** (Sidecar-Stil) regeln:

    * `productpage → details` (GET)
    * `reviews → ratings` (GET/POST)
    * `gateway → productpage` (L4, kein Waypoint nötig)

* **Endzustand**:

  * Istio läuft im **Ambient-Mode** (Profil `ambient`).
  * Namespace `bookinfo` ist mit `istio.io/dataplane-mode=ambient` gelabelt.
  * **Keine Sidecars** mehr, nur ztunnel + Waypoint.
  * L7-Policies werden über **Waypoint + `targetRefs`** durchgesetzt.

### A2. Technische Voraussetzungen

* Laufender Kubernetes-Cluster (kind, k3d, Minikube, EKS, GKE, …).
* `kubectl` konfiguriert.
* `curl`, `bash`.
* Internetzugang aus der Shell, um Istio & CRDs zu holen.

```bash
kubectl cluster-info
kubectl get nodes
```

**Verifikation:**

* Mindestens ein Node im Status `Ready`.

---

## B. Arbeitsverzeichnis & Istio herunterladen

### B1. Arbeitsverzeichnis für Manifeste

Alle eigenen Dateien landen hier:

```bash
mkdir -p ~/manifests/migrate-from-sidecar-to-ambient
cd ~/manifests/migrate-from-sidecar-to-ambient
```

**Verifikation:**

```bash
pwd
# sollte mit /migrate-from-sidecar-to-ambient enden
```

---

### B2. Istio herunterladen

```bash
cd ~
curl -L https://istio.io/downloadIstio | sh -

cd "$(ls -d istio-* | head -n1)"
export PATH="$PWD/bin:$PATH"

istioctl version
```

**Verifikation:**

* `istioctl version` zeigt mindestens `client version`.

---

## C. Sidecar-Setup (Ausgangssituation)

Wir bauen die **Sidecar-Welt** nach, von der im Video gesprochen wird.

---

### C1. Istio (Sidecar) installieren – demo-Profil

```bash
cd ~/istio-*/
export PATH="$PWD/bin:$PATH"

istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y
```

**Verifikation:**

```bash
kubectl get pods -n istio-system
```

* `istiod-...` und weitere Core-Pods im Status `Running`.

---

### C2. Namespace `bookinfo` anlegen & für Sidecar-Injection labeln

```bash
kubectl create namespace bookinfo

kubectl label namespace bookinfo istio-injection=enabled --overwrite
kubectl get ns bookinfo --show-labels
```

**Verifikation:**

* Label `istio-injection=enabled` sichtbar.

---

### C3. Gateway API CRDs installieren

```bash
kubectl get crd gateways.gateway.networking.k8s.io &>/dev/null || \
  kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.4.0" \
    | kubectl apply -f -
```

**Verifikation:**

```bash
kubectl get crd gateways.gateway.networking.k8s.io
```

---

### C4. Bookinfo-App deployen (Sidecar-Basis)

```bash
cd ~/istio-*/

kubectl apply -n bookinfo -f samples/bookinfo/platform/kube/bookinfo.yaml

kubectl get pods -n bookinfo
kubectl get svc  -n bookinfo
```

**Verifikation:**

* Pods wie `productpage-...`, `details-...`, `reviews-...`, `ratings-...`.
* `READY` sollte `2/2` sein (App + Sidecar).

---

### C5. Gateway + HTTPRoute für `productpage`

```bash
cd ~/istio-*/

kubectl apply -n bookinfo -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml

kubectl wait --for=condition=programmed gtw bookinfo-gateway -n bookinfo --timeout=120s

kubectl get gtw -n bookinfo
kubectl get svc -n bookinfo | grep bookinfo-gateway
```

Port-Forward für externen Zugriff:

```bash
kubectl -n bookinfo port-forward svc/bookinfo-gateway-istio 8080:80
```

Im Browser:
`http://localhost:8080/productpage`

**Verifikation (im Cluster):**

```bash
kubectl -n bookinfo exec \
  "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- \
  curl -sS productpage:9080/productpage | grep "<title>"
```

* `<title>Simple Bookstore App</title>` sollte erscheinen.

---

## D. Sidecar-AuthorizationPolicies (Ist-Zustand)

Wir bilden die Policies nach, die im Transcript beschrieben werden:

* **Gateway → productpage** (L4)
* **productpage → details** (L7 – GET)
* **reviews → ratings** (L7 – GET/POST)

> Für die Sidecar-Welt nutzen wir klassische `selector`-basierte Policies.

---

### D1. Gateway-Policy (L4, kein Waypoint nötig)

Datei:

```bash
cd ~/manifests/migrate-from-sidecar-to-ambient

cat > sidecar-authz-gateway-productpage.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-viewer
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-gateway-istio"
EOF
```

Anwenden:

```bash
kubectl apply -f sidecar-authz-gateway-productpage.yaml
```

**Verifikation:**

```bash
kubectl get authorizationpolicy -n bookinfo
kubectl describe authorizationpolicy productpage-viewer -n bookinfo
```

---

### D2. Details-Policy (productpage → details, nur GET)

```bash
cat > sidecar-authz-details.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: details-policy
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-productpage"
    to:
    - operation:
        methods: ["GET"]
EOF
```

```bash
kubectl apply -f sidecar-authz-details.yaml
```

---

### D3. Ratings-Policy (reviews → ratings, GET/POST)

```bash
cat > sidecar-authz-ratings.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ratings-policy
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-reviews"
    to:
    - operation:
        methods: ["GET","POST"]
EOF
```

```bash
kubectl apply -f sidecar-authz-ratings.yaml
```

---

### D4. Verifikation der Sidecar-Policies

`sleep`-Beispiel als Test-Client:

```bash
kubectl apply -n bookinfo -f samples/sleep/sleep.yaml
kubectl get pod -n bookinfo -l app=sleep
```

Optionaler Test:

```bash
# Zugriff vom sleep-Pod auf productpage -> sollte durch Gateway-Policy abgelehnt sein
kubectl -n bookinfo exec deploy/sleep -- \
  curl -sS http://productpage:9080/productpage || echo "DENIED as expected"
```

---

## E. Ambient-Komponenten installieren

### E1. Ambient-Profil installieren

```bash
cd ~/istio-*/
export PATH="$PWD/bin:$PATH"

istioctl install --set profile=ambient --skip-confirmation
```

**Verifikation:**

```bash
kubectl get pods -n istio-system
```

* `ztunnel-...`-Pods auf allen Nodes (DaemonSet).

---

### E2. Bestehende Sidecar-Injection deaktivieren

```bash
kubectl label namespace bookinfo istio-injection- || true
kubectl get ns bookinfo --show-labels
```

**Verifikation:**

* `istio-injection`-Label ist nicht mehr vorhanden.

---

### E3. Namespace in Ambient-Mesh aufnehmen

```bash
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient --overwrite
kubectl get ns bookinfo --show-labels
```

**Verifikation:**

* Label `istio.io/dataplane-mode=ambient` vorhanden.

> Bis hierhin: Workloads haben **noch Sidecars**; Ambient ist installiert, aber noch nicht aktiv für die Pods.

---

## F. Waypoint-Proxy bereitstellen (L7-Funktionen)

Workloads mit L7-Policies (details, ratings) brauchen einen **Waypoint**.

### F1. Namespace-Waypoint erzeugen & einschreiben

```bash
istioctl waypoint apply -n bookinfo --enroll-namespace --wait
```

**Verifikation:**

```bash
kubectl get gtw -n bookinfo
kubectl get pods -n bookinfo -l istio.io/gateway-name=waypoint
```

* Gateway `waypoint` ist `PROGRAMMED=True`.
* Ein Waypoint-Pod läuft im Namespace.

---

## G. Ambient-kompatible AuthorizationPolicies erstellen

**Wichtiger Unterschied:**

* Sidecar-Policies: `selector` (workload-basiert).
* Ambient/Waypoint-Policies: **`targetRefs`** (service-basiert).

Wir lassen die **alten Policies** erst einmal stehen und legen zusätzliche Ambient-Policies an.

---

### G1. L4-Policy für `productpage` (gateway → productpage, ztunnel)

```bash
cd ~/manifests/migrate-from-sidecar-to-ambient

cat > ambient-authz-productpage-l4.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-ztunnel
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-gateway-istio"
EOF
```

```bash
kubectl apply -f ambient-authz-productpage-l4.yaml
kubectl get authorizationpolicy -n bookinfo
```

---

### G2. L7-Policy für `details` (productpage → details, GET) – Waypoint

```bash
cat > ambient-authz-details-l7-waypoint.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: details-waypoint
  namespace: bookinfo
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-productpage"
    to:
    - operation:
        methods: ["GET"]
EOF
```

```bash
kubectl apply -f ambient-authz-details-l7-waypoint.yaml
```

---

### G3. L7-Policy für `ratings` (reviews → ratings, GET/POST) – Waypoint

```bash
cat > ambient-authz-ratings-l7-waypoint.yaml << 'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: ratings-waypoint
  namespace: bookinfo
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-reviews"
    to:
    - operation:
        methods: ["GET","POST"]
EOF
```

```bash
kubectl apply -f ambient-authz-ratings-l7-waypoint.yaml
kubectl get authorizationpolicy -n bookinfo
```

**Verifikation:**

```bash
kubectl describe authorizationpolicy details-waypoint -n bookinfo
kubectl describe authorizationpolicy ratings-waypoint -n bookinfo
```

* `targetRefs` sind sichtbar.

---

## H. Workloads von Sidecar → Ambient migrieren (Rollout)

### H1. Rollout für Bookinfo-Workloads

```bash
kubectl rollout restart deploy -n bookinfo
kubectl get pods -n bookinfo
```

**Verifikation:**

```bash
kubectl get pods -n bookinfo \
  -o=jsonpath='{range .items[*]}{.metadata.name}{" => "}{.status.containerStatuses[*].name}{"\n"}{end}'
```

* Nur noch **ein Container pro Pod**, kein `istio-proxy`.

---

### H2. Ambient-Mitgliedschaft prüfen

```bash
istioctl x workload list -n bookinfo
```

* Workloads sollten als Ambient-Workloads sichtbar sein.

---

## I. Tests der neuen Ambient-Policies

Wir testen analog zum Video:

* Sleep-Client darf `productpage` nicht erreichen.
* Sleep-Client darf `details` und `ratings` nicht erreichen.
* Gateway- und interne Service-Kommunikation (Bookinfo-Flow) funktionieren weiter.

---

### I1. Sleep-Pod sicherstellen

Falls nicht vorhanden:

```bash
kubectl apply -n bookinfo -f samples/sleep/sleep.yaml
kubectl get pod -n bookinfo -l app=sleep
```

---

### I2. Gateway-Policy: Sleep darf `productpage` NICHT erreichen

```bash
kubectl -n bookinfo exec deploy/sleep -- \
  curl -sS http://productpage:9080/productpage || echo "DENIED as expected"
```

**Erwartung:**

* 403 / Verbindungsfehler – Sleep ist blockiert.

Der Gateway-Zugriff (Browser via Port-Forward) sollte weiterhin funktionieren.

---

### I3. Details-Policy: Sleep → details verboten

```bash
kubectl -n bookinfo exec deploy/sleep -- \
  curl -sS http://details:9080/details/1 || echo "DENIED as expected"
```

**Erwartung:**

* 403 / Permission denied.

---

### I4. Ratings-Policy: Sleep → ratings verboten

```bash
kubectl -n bookinfo exec deploy/sleep -- \
  curl -sS http://ratings:9080/ratings/1 || echo "DENIED as expected"
```

**Erwartung:**

* 403 / Permission denied.

---

## J. Aufräumen der „Legacy“-Sidecar-Policies

Wenn die Ambient-Policies korrekt funktionieren, können die alten Selector-basierten Policies entfernt werden.

```bash
kubectl delete authorizationpolicy details-policy  -n bookinfo --ignore-not-found
kubectl delete authorizationpolicy ratings-policy  -n bookinfo --ignore-not-found
# Optional: Gateway-Policy der Sidecar-Welt entfernen
# kubectl delete authorizationpolicy productpage-viewer -n bookinfo --ignore-not-found
```

**Verifikation:**

```bash
kubectl get authorizationpolicy -n bookinfo
```

* Es sollten nur noch die Ambient-Policies übrig sein:

  * `productpage-ztunnel`
  * `details-waypoint`
  * `ratings-waypoint`

---

## K. Troubleshooting – typische Fallen Sidecar → Ambient

### 1. Pods haben weiterhin Sidecars

**Symptom:**

* `READY 2/2`, Container `istio-proxy` noch da.

**Ursache:**

* `istio-injection=enabled` Label nicht entfernt oder Deployments nicht neu gestartet.

**Lösung:**

```bash
kubectl label ns bookinfo istio-injection- || true
kubectl rollout restart deploy -n bookinfo
```

---

### 2. Ambient wirkt gar nicht (kein mTLS / kein ztunnel)

**Symptom:**

* `istioctl x workload list -n bookinfo` leer.
* Traffic wirkt wie „plain Kubernetes“.

**Ursache:**

* Namespace nicht mit `istio.io/dataplane-mode=ambient` gelabelt.

**Lösung:**

```bash
kubectl label ns bookinfo istio.io/dataplane-mode=ambient --overwrite
kubectl rollout restart deploy -n bookinfo
```

---

### 3. L7-Policies greifen nicht

**Symptom:**

* `AuthorizationPolicy` mit `targetRefs` existiert, aber Requests werden nicht gefiltert.

**Mögliche Ursachen:**

* Kein Waypoint im Namespace.
* Workloads nutzen den Waypoint nicht.

**Lösung:**

```bash
istioctl waypoint apply -n bookinfo --enroll-namespace --wait
kubectl get gtw -n bookinfo
kubectl get pod -n bookinfo -l istio.io/gateway-name=waypoint
kubectl rollout restart deploy -n bookinfo
```

---

### 4. Alles ist verboten (403 überall)

**Symptom:**

* Auch eigentlich erlaubte Pfade/Clients bekommen 403.

**Typische Ursachen:**

* L4-Policy (ztunnel) blockiert Waypoint oder Gateway.
* Falsche `principals` (ServiceAccount-Namen stimmen nicht).

**Checks:**

```bash
kubectl get sa -n bookinfo
kubectl get authorizationpolicy -n bookinfo -o yaml
```

**Lösungen:**

* `principals` an tatsächliche ServiceAccount-Namen anpassen.
* Ggf. zusätzliche Principals (z. B. Waypoint-ServiceAccount) erlauben.

---

### 5. Gateway API „no matches for kind Gateway“

**Symptom:**

* Beim Anwenden von Gateway-Manifesten kommt `no matches for kind "Gateway"`.

**Ursache:**

* Gateway-CRDs nicht installiert.

**Lösung:**

```bash
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.4.0" \
  | kubectl apply -f -
```

