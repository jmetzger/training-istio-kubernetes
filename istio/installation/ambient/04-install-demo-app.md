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

```
kubectl describe namespace bookinfo
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

Auf den Trainings-Clustern läuft MetalLB. Der Service des Gateways bekommt deshalb automatisch eine
`EXTERNAL-IP` (die öffentliche IP eines Workers), und das Gateway zeigt `PROGRAMMED True`:

```
kubectl -n bookinfo get gateways
kubectl -n bookinfo get svc bookinfo-gateway-istio
```

Direkt nach dem Anlegen steht das Gateway noch kurz auf `PROGRAMMED False` und die `EXTERNAL-IP` auf `<pending>`.
Nach wenigen Sekunden (Befehle wiederholen) steht dort `True` und die IP.

```
# IP des Gateways merken
GATEWAY_IP=$(kubectl -n bookinfo get svc bookinfo-gateway-istio -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP
```

```
curl -s http://$GATEWAY_IP/productpage | grep -o "<title>.*</title>"
```

Im Browser: `http://<GATEWAY_IP>/productpage`

Hinweise:

  * Die IP ist öffentlich erreichbar, auch von eurem eigenen Rechner aus.
  * Pro Cluster gibt es zwei IPs (die beiden Worker), also höchstens **zwei** Services vom Typ `LoadBalancer`
    gleichzeitig. Ein weiterer bleibt auf `<pending>`. Nicht mehr benötigte Gateways oder Services vorher löschen.

## Ref

  * https://istio.io/latest/docs/ambient/getting-started/deploy-sample-app/
