# Istio Installation mit Helm (mit demo-profile)

## Voraussetzungen

- Kubernetes-Cluster läuft und `kubectl` ist konfiguriert
- Helm 3 ist installiert (`helm version`)

## Profile 

  * Die Charts von istio verwenden as profile Konzept
  * Diese werden nachher einfach mit dem defaults - Block gemerged
  * Wenn wir z.B. eine Einstellungen für die Demo haben wollen (z.B. sampleTracing auf 100 statt auf 1), können wir das demo-profile verwenden
  * Die Profile für die Helm-Charts findest Du hier (nicht alles, was es für die Installation mit istioctl gibt, gibt es auch mit helm !!): 
    [Profile für Helm]https://github.com/istio/istio/tree/release-1.29.1/manifests/helm-profiles)
  * Wichtig ! Sie werden nicht in allen Charts verwenden, wenn wir es überall angeben, müssen wir uns aber keine Gedanken machen, in welchem Start 


## Schritt 1: Helm-Repository hinzufügen

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

## Schritt 2: Namespace erstellen und Base-Chart installieren

Das Base-Chart installiert die Istio CRDs (Custom Resource Definitions):

```bash
helm install istio-base istio/base -n istio-system --set defaultRevision=default --set profile=demo --create-namespace --version 1.29.1
```

Prüfen:

```bash
helm ls -n istio-system
```

Der Status von `istio-base` sollte `deployed` sein.

## Schritt 3: Istiod (Control Plane) installieren

```bash
helm install istiod istio/istiod -n istio-system --wait --version 1.29.1  --set profile=demo
```

Prüfen:

```bash
helm ls -n istio-system
kubectl get deployments -n istio-system
```

`istiod` sollte `READY 1/1` anzeigen.

## Schritt 4 : Ingress Gateway installieren

```bash
helm install istio-ingress istio/gateway -n istio-ingress --wait --version 1.29.1 --set profile=demo --create-namespace
```

> **Hinweis:** Der Gateway-Namespace darf kein Label `istio-injection=disabled` haben.

## Schritt 4.5: (Optional) Metrics-Server installieren 

  * Der HorizontalPodAutoscaler der vom Gateway - Chart verwendet wird braucht den Metrics-Server um die CPU vom istio-ingressgateway - Pod auszulesen
  * kubectl -n istio-ingress get hpa # Hier steht aktuell **Unknown**

<img width="1527" height="94" alt="image" src="https://github.com/user-attachments/assets/9000a286-3407-452a-8092-43882df72817" />

  * Um das zu beheben müssen wir den Metrics-Server installieren




```
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
```


```
cd 
mkdir -p helm-charts/metrics-server
cd helm-charts/metrics-server
nano values.yaml
```

```
args:
  - --kubelet-insecure-tls
```



```
helm -n kube-system upgrade --install metrics-server metrics-server/metrics-server --version 3.13.0 --create-namespace -f values.yaml 
```

```
# Überprüfen, es dauert ein bisschen, bis er Ready
kubectl -n kube-system get pods
```

```
# hpa sollte jetzt die richtigen Infos haben
kubectl -n istio-ingress get hpa 
```

<img width="1408" height="95" alt="image" src="https://github.com/user-attachments/assets/192af4aa-a741-4d58-bc9c-c65f04b1395f" />


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
