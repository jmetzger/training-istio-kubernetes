# Istio Installation mit Helm (mit demo-profile)

## Voraussetzungen

- Kubernetes-Cluster läuft und `kubectl` ist konfiguriert
- Helm 3 ist installiert (`helm version`)

## Schritt 1: Helm-Repository hinzufügen

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

## Schritt 2: Namespace erstellen und Base-Chart installieren

Das Base-Chart installiert die Istio CRDs (Custom Resource Definitions):

```bash
helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
```

Prüfen:

```bash
helm ls -n istio-system
```

Der Status von `istio-base` sollte `deployed` sein.

## Schritt 3: Istiod (Control Plane) installieren

```bash
helm install istiod istio/istiod -n istio-system --wait
```

Prüfen:

```bash
helm ls -n istio-system
kubectl get deployments -n istio-system
```

`istiod` sollte `READY 1/1` anzeigen.

## Schritt 4 (Optional): Ingress Gateway installieren

```bash
kubectl create namespace istio-ingress
helm install istio-ingress istio/gateway -n istio-ingress --wait
```

> **Hinweis:** Der Gateway-Namespace darf kein Label `istio-injection=disabled` haben.

## Konfiguration anpassen

Default-Werte eines Charts anzeigen:

```bash
helm show values istio/istiod
helm show values istio/gateway
```

Eigene Werte beim Installieren mitgeben:

```bash
helm install istiod istio/istiod -n istio-system --values my-values.yaml --wait
```

## Deinstallation

In umgekehrter Reihenfolge deinstallieren:

```bash
# 1. Gateway (falls installiert)
helm delete istio-ingress -n istio-ingress
kubectl delete namespace istio-ingress

# 2. Istiod
helm delete istiod -n istio-system

# 3. Base-Chart
helm delete istio-base -n istio-system

# 4. Namespace aufräumen
kubectl delete namespace istio-system
```

Optional – CRDs entfernen (Achtung: löscht alle Istio-Ressourcen!):

```bash
kubectl get crd -oname | grep 'istio.io' | xargs kubectl delete
```

## Referenz

- [Offizielle Doku](https://istio.io/latest/docs/setup/install/helm/)
- [Helm Upgrade Guide](https://istio.io/latest/docs/setup/upgrade/helm/)
- [Advanced Helm Customization](https://istio.io/latest/docs/setup/additional-setup/customize-installation-helm/)
