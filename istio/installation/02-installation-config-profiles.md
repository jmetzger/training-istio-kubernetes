# Installatios-Konfigurations-Profile 

  * istio verwendet verschiedene vorgefertigte Profile, die das Ausrollen (installieren) erleichtern
  * Diese können in istioctl verwendet werden, aber auch mit dem helm-chart

## Welcher Profile gibt es ? 

  * Es gibt deployment - profile und platform profile

## Übersicht der Deployment - Profile 

| Core components        | default | demo | minimal | remote | empty | preview | ambient |
|------------------------|---------|------|---------|--------|-------|---------|---------|
| istio-egressgateway    |         | ✓    |         |        |       |         |         |
| istio-ingressgateway   | ✓       | ✓   |         |        |       | ✓       |         |
| istiod                 | ✓       | ✓    | ✓       |        |       | ✓       | ✓       |
| CNI                    |         |      |         |        |       |         | ✓       |
| Ztunnel                |         |      |         |        |       |         | ✓       |

## Welches Deployment - Profile nehme ich am besten 

  * Für Productiion am besten Default (mit Sidecar) oder Ambient (für den Ambient - Modus)
  * Zum Testen / Üben demo (mit Sidecar), aber nicht ! für die Production. Schlecht für Performance (Hier ist ganz viel Debuggen und Tracing aktiviert)

## Übersicht Platform - Profile 

| Plattform | Beschreibung |
|-----------|--------------|
| gke | Sets chart options required or recommended for installing Istio in Google Kubernetes Engine (GKE) environments. |
| eks | Sets chart options required or recommended for installing Istio in Amazon's Elastic Kubernetes Service (EKS) environments. |
| openshift | Sets chart options required or recommended for installing Istio in OpenShift environments. |
| k3d | Sets chart options required or recommended for installing Istio in k3d environments. |
| k3s | Sets chart options required or recommended for installing Istio in K3s environments. |
| microk8s | Sets chart options required or recommended for installing Istio in MicroK8s environments. |
| minikube | Sets chart options required or recommended for installing Istio in minikube environments. |

## Reference:

  * https://istio.io/latest/docs/setup/additional-setup/config-profiles/
