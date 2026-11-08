# Demo - App Installation (Ambient / Waypoint Proxy)

## Überblick

<img width="693" height="465" alt="image" src="https://github.com/user-attachments/assets/22cbf386-5a90-458b-8157-51620ef829ea" />

## (Optional) Alte Installation entfernen 

```
kubectl delete ns bookinfo
```

## Vorbereitung

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

## Waypoint Proxy ausrollen

```
cd
mkdir -p manifests/waypoint
cd manifests/waypoint
```  


```
# YAML generieren (dry-run)
istioctl waypoint generate --namespace bookinfo --for all  > waypoint.yaml 
```

```
# Anschauen was passiert
cat waypoint.yaml 
```

```
# Ausrollen
kubectl apply -f waypoint.yaml
kubectl label namespace bookinfo istio.io/use-waypoint=waypoint
```

```
# Überprüfen
kubectl -n bookinfo get gateways
istioctl waypoint list --namespace bookinfo
```

## Bookinfo App ausrollen

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/platform/kube/bookinfo-versions.yaml
kubectl -n bookinfo get all
```

## Testen ob die App funktioniert

```
kubectl -n bookinfo exec "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

```
# Einfachere Variante
kubectl -n bookinfo exec deployments/ratings-v1 -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

## App mit Gateway API nach aussen öffnen

```
# That's what we do ....
cat ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
```

```
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl -n bookinfo get gateways
kubectl -n bookinfo get httproutes -o yaml
```

```
# Das Gateway erstellt automatisch einen Service
kubectl -n bookinfo get svc bookinfo-gateway-istio
```

### Zugriff auf die App

Die Trainings-Cluster haben keinen LoadBalancer (kein MetalLB, keine Cloud-Anbindung).
Die `EXTERNAL-IP` des Service bleibt deshalb auf `<pending>` und das Gateway zeigt `PROGRAMMED False`.
Die App ist trotzdem erreichbar:

```
# Variante 1: port-forward (läuft im Vordergrund, in einem zweiten Terminal testen)
kubectl -n bookinfo port-forward svc/bookinfo-gateway-istio 8080:80
```

```
curl -s http://localhost:8080/productpage | grep -o "<title>.*</title>"
```

```
# Variante 2: NodePort auf der Node-IP (vom Client aus erreichbar)
NODEPORT=$(kubectl -n bookinfo get svc bookinfo-gateway-istio -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}')
NODEIP=$(kubectl get nodes -o jsonpath='{.items[1].status.addresses[?(@.type=="InternalIP")].address}')
curl -s http://$NODEIP:$NODEPORT/productpage | grep -o "<title>.*</title>"
```

Mit einem LoadBalancer (z.B. in einem Cloud-Cluster) gilt: die `EXTERNAL-IP` notieren und
`http://<external-ip>/productpage` aufrufen.

## Ref

  * https://istio.io/latest/docs/ambient/getting-started/deploy-sample-app/
