# Debug

## Why like this ?

 * We need a way that a service is mounted into the pod (service-account is used) 
 * Same serviceAccount that would be used productpage - pod itself 

## Step 1: Debug-Container zum Debuggen 

  * Debug Container in productpage - pod starten, um Verbindung zu pod -> Review zu debuggen 

```
kubectl -n bookinfo get pods | grep productpage
# diesen entsprechend hier verwenden 
kubectl -n bookinfo debug productpage-v1-54bb874995-rr7cv -it --image=busybox
```

```
# in der bash 
wget -O - http://reviews:9080/reviews/1
exit
```

>[NOTE:]
> Achtung ! Danach 1-2 Minuten warten !!! 
```
# AuthorizationPolicy rausnehmen
kubectl delete -f 03-reviews-from-productpage.yaml
```

```
kubectl -n bookinfo debug productpage-v1-54bb874995-rr7cv -it --image=busybox
```

```
# in der bash 
wget -O - http://reviews:9080/reviews/1
```

```
exit
```

## Step 2(V2 - Variante) 

```
kubectl -n bookinfo run -it podtester --image=busybox --overrides='{ "spec": { "serviceAccount": "bookinfo-productpage" }  }'
```
