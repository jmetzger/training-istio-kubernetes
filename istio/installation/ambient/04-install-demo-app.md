# Demo - App Installation (Ambient / Waypoint Proxy)

## Überblick

<img width="693" height="465" alt="image" src="https://github.com/user-attachments/assets/22cbf386-5a90-458b-8157-51620ef829ea" />

## Vorbereitung

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
```

## Waypoint Proxy ausrollen

```
# YAML generieren (dry-run)
istioctl waypoint apply --namespace bookinfo --enroll-namespace --mode all --dry-run -o yaml > /tmp/waypoint.yaml
```

```
# Anschauen was passiert
nano /tmp/waypoint.yaml
```

```
# Ausrollen
kubectl apply -f /tmp/waypoint.yaml
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
# Die external-ip aus diesem Output notieren
# Das Gateway erstellt automatisch einen Service
kubectl -n bookinfo get svc bookinfo-gateway-istio
```

```
http://<external-ip>/productpage
# oder im Browser öffnen
```

## Ref

  * https://istio.io/latest/docs/ambient/getting-started/deploy-sample-app/
