# Demo - app installation

## Überblick 

<img width="992" height="615" alt="image" src="https://github.com/user-attachments/assets/5773ce04-fd83-45a6-9914-d2b1b72c1505" />

## Vorbereitung

### Namespace erstellen und labeln. 
```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

## bookdemo app ausrollen 


```
kubectl -n bookinfo apply -f  ~/istio/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl -n bookinfo get all 
```

## testen ob die app funktioniert 

```
kubectl -n bookinfo exec "$(kubectl -n bookinfo get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

## App mit istio-ingress-gateway nach aussen öffnen 

```
# That's what we do ....
kubectl -n bookinfo apply -fca



```

```
cd
mkdir -p manifests/istio-gateway
cd manifests/istio-gateway
nano gateway.yaml
```

```
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  # The selector matches the ingress gateway pod labels.
  # If you installed Istio using Helm following the standard documentation, this would be "istio=ingress"
  selector:
    istio: ingress # use istio default controller
  servers:
  - port:
      number: 80
      name: http2
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

```
kubectl apply -f .
```



```
kubectl -n bookinfo get gw
kubectl -n bookinfo get virtualservice -o yaml 
```

```
# get ip from here
kubectl -n istio-ingress get all
kubectl -n istio-ingress get svc 
```

```
http://<external-ip>/productpage 
# or in your browser
```


