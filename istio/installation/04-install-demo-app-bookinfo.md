# Demo - app installation

## Überblick 

<img width="992" height="615" alt="image" src="https://github.com/user-attachments/assets/5773ce04-fd83-45a6-9914-d2b1b72c1505" />

## Vorbereitung

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

## bookdemo app ausrollen 


```
kubectl -n bookinfo apply -f  ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo get all 
```
