# Demo - App Installation (Ambient / Waypoint Proxy)

## Überblick

<img width="693" height="465" alt="image" src="https://github.com/user-attachments/assets/22cbf386-5a90-458b-8157-51620ef829ea" />

## Prerequisites 

  * bookinfo ist installiert im modus sidecar 

## Vorbereitung

```
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient
kubectl label namespace bookinfo istio-injection-
kubectl describe ns bookinfo 
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
kubectl -n bookinfo rollout restart deployment
kubectl -n bookinfo get pods 
```

## Testen ob die App funktioniert

```
kubectl -n bookinfo exec "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

```
# Einfachere Variante
kubectl -n bookinfo exec deployments/ratings-v1 -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

## Von aussen Anfragen über gateway (gateway api)

```
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

## Testen aus einem namespace der nicht teil des istio mesh ist 

```
kubectl create ns no-mesh
kubectl -n no-mesh apply -f https://raw.githubusercontent.com/istio/istio/release-1.25/samples/sleep/sleep.yaml
kubectl -n no-mesh wait --for=condition=ready pod --all --timeout=60s

# Zugriff auf productpage versuchen
kubectl -n no-mesh exec deploy/sleep -- \
  curl -s -o /dev/null -w "%{http_code}\n" productpage.bookinfo:9080/productpage
```


## Ref

  * https://istio.io/latest/docs/ambient/getting-started/deploy-sample-app/
