# Demo - app installation

## Vorbereitung

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

## bookdemo app ausrollen 


```
kubectl -n bookinfo apply -f  ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
```
