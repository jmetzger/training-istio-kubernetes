# Ingress Istio-Gateway vs. Kubernetes Gateway API 

## Istio Ingress-Gateway-Flow (bookinfo/productpage) 

![Flow](/images/screenshot_20260314_142715.png)

## Gateway API -> Gateway Flow (bookinfo/productpage)

![Flow](/images/istio/gateway-api-flow.png)

 * Achtung:  VirtualService und DestinationRule werden teilweise immer noch benötigt, auch wenn man mit der Gateway API arbeitet,
     um bestimmte Feature abzubilden  

## Schema: Istio Ingress-Gateway

![Istio Ingress-Gateway: Datenpfad und Konfiguration](../../images/istio/ingress-istio-gateway.svg)

## Schema: Kubernetes Gateway API

![Kubernetes Gateway API: Datenpfad und Konfiguration](../../images/istio/ingress-gateway-api.svg)
