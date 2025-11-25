# We want show the logs of istio proxy to identify AuthorizationPolicy 

## Schritt 1: Logs einschalten 

```
 POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
 kubectl exec -it -n bookinfo "$POD" -c istio-proxy -- curl -X POST "localhost:15000/logging?rbac=debug"
 kubectl logs "$POD" -n bookinfo -c istio-proxy -f
```

## Schritt 2: Browseraufruf 

```
```

## Schritt 3: Logs analysieren (Ausgabe aus Schritt 1) 


## Schritt 4: Logs abschalten 

``` 
kubectl exec -it -n bookinfo "$POD" -c istio-proxy -- curl -X POST "localhost:15000/logging?rbac=info"
```
