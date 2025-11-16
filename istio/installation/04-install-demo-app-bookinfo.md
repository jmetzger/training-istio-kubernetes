# Demo - app installation

## Vorbereitung

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

## Was installieren wir ? 





## bookdemo app ausrollen 


```
kubectl -n bookinfo -f ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
```
