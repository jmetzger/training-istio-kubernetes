# Istio CNI Plugin — Installation im Rahmen eines istiod Upgrades

## Warum das Istio CNI Plugin?

Das Istio CNI Plugin ersetzt den `istio-init` Init-Container, der normalerweise die iptables-Regeln für die Traffic-Umleitung zum Envoy Sidecar setzt.

### Vorteile

- **Keine erhöhten Privileges im Pod** — `NET_ADMIN` und `NET_RAW` Capabilities entfallen, da die iptables-Regeln auf Node-Ebene gesetzt werden
- **Kompatibel mit `restricted` Pod Security Standard** — wichtig für sicherheitskritische Umgebungen
- **Keine Race Conditions** — Traffic-Regeln stehen bereits bevor der erste Container im Pod startet. Ohne CNI Plugin können Init-Container, die vor `istio-init` laufen, Traffic am Envoy vorbeischicken (kein mTLS, keine AuthorizationPolicy)
- **Weniger Ressourcen** — kein zusätzlicher Init-Container-Image-Pull
- **Im Ambient Mode zwingend erforderlich** — wer später auf Ambient Mode migrieren möchte, braucht das CNI Plugin ohnehin

## Voraussetzungen

- Bestehende Istio-Installation via Helm (base + istiod) mit `profile=demo`
- Helm >= 3.x
- `kubectl` Zugriff auf den Cluster
- Das CNI Plugin arbeitet als **Chained CNI Plugin** — es hängt sich an das bestehende CNI an (Flannel, Calico etc.). Bei Cilium-basierten Clustern (z.B. DigitalOcean DOKS, GKE Dataplane v2) vorher Kompatibilität prüfen.

## Schritt-für-Schritt-Anleitung

### 1. Aktuelle Installation prüfen

```bash
# Installierte Istio Helm Releases anzeigen
helm list -n istio-system

# Aktuelle istiod-Werte prüfen
helm get values istiod -n istio-system
```

Verifiziere, dass bestehende Pods einen `istio-init` Init-Container haben:

```bash
# alle pods im bookinfo - namespace 
kubectl -n bookinfo get pods details-v1-6cc9f5cc44-c5zqf -o jsonpath='{.spec.initContainers[*].name}'
```

Erwartete Ausgabe (vor CNI Plugin): `istio-init`

### 2. Istio Helm Repo aktualisieren

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### 3. Istio CNI Plugin installieren

```bash
helm install istio-cni istio/cni -n istio-system \
  --set profile=demo
```

Warten bis das CNI DaemonSet auf allen Nodes läuft:

```bash
kubectl rollout status daemonset/istio-cni-node -n istio-system
```

### 4. istiod upgraden — CNI Plugin aktivieren

```bash
helm upgrade istiod istio/istiod -n istio-system \
  --set profile=demo \
  --set pilot.cni.enabled=true
```

Prüfen, ob istiod den Neustart sauber durchgeführt hat:

```bash
kubectl rollout status deployment/istiod -n istio-system
```

### 5. Bestehende Workloads neu starten

Bestehende Pods haben noch den alten `istio-init` Init-Container. Erst nach einem Restart werden sie ohne Init-Container und mit CNI Plugin Traffic-Umleitung gestartet.

```bash
# Alle Deployments in einem Namespace neu starten
kubectl rollout restart deployment -n <NAMESPACE>

# Oder gezielt ein einzelnes Deployment
kubectl rollout restart deployment/<DEPLOYMENT_NAME> -n <NAMESPACE>
```

### 6. Verifizierung

**CNI DaemonSet läuft auf allen Nodes:**

```bash
kubectl get daemonset istio-cni-node -n istio-system
```

**Neue Pods haben keinen `istio-init` Init-Container mehr:**

```bash
kubectl get pod <NEUER_POD> -n <NAMESPACE> -o jsonpath='{.spec.initContainers[*].name}'
```

Erwartete Ausgabe: leer (kein `istio-init`)

**Sidecar ist weiterhin injiziert:**

```bash
kubectl get pod <NEUER_POD> -n <NAMESPACE> -o jsonpath='{.spec.containers[*].name}'
```

Erwartete Ausgabe: `<APP_CONTAINER> istio-proxy`

**Traffic funktioniert weiterhin:**

```bash
# Aus einem Pod heraus testen
kubectl exec <POD> -c <APP_CONTAINER> -- curl -s http://<SERVICE>.<NAMESPACE>.svc.cluster.local
```

## Troubleshooting

### CNI DaemonSet startet nicht

```bash
kubectl describe daemonset istio-cni-node -n istio-system
kubectl logs -l k8s-app=istio-cni-node -n istio-system
```

Häufige Ursache: CNI-Bin- oder Conf-Verzeichnis stimmt nicht mit dem Cluster-Setup überein. Kann über Helm Values angepasst werden:

```bash
helm upgrade istio-cni istio/cni -n istio-system \
  --set profile=demo \
  --set cni.cniBinDir=/opt/cni/bin \
  --set cni.cniConfDir=/etc/cni/net.d
```

### Pods haben nach Restart immer noch `istio-init`

Prüfen, ob `pilot.cni.enabled=true` gesetzt ist:

```bash
helm get values istiod -n istio-system | grep cni
```

### Netzwerk-Konnektivität unterbrochen nach CNI Installation

Rollback möglich:

```bash
helm upgrade istiod istio/istiod -n istio-system \
  --set profile=demo \
  --set pilot.cni.enabled=false

kubectl rollout restart deployment -n <NAMESPACE>

helm uninstall istio-cni -n istio-system
```
