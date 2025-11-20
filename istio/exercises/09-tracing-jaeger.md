# Tracing with jaeger 

## Prerequisites 

  * Jaeger is setup
  * demo profile is set up
  * bookinfo is rolled out

## Note: 

  * You do NOT need to enable tracing, because it is already active in demo profile
  * You need so set up a telemetry - object

## Walktrough 

```
cd
mkdir -p manifests/tracing-jaeger
cd manifests/tracing-jaeger
```

```
nano 01-telemetry-jaeger.yaml 
```

```
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
```

```
kubectl apply -f .
```

## Test it: 

```
# Adjust to your own IP 
GATEWAY_URL=164.90.237.35
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage";
```

## Open jaeger.tlnxx.do.t3isp.de 

```
Search ->
```

<img width="436" height="147" alt="image" src="https://github.com/user-attachments/assets/0bb5069b-ee3c-4799-91fa-35b498a3c047" />

<img width="444" height="61" alt="image" src="https://github.com/user-attachments/assets/97451972-7070-46ab-a4e4-39dc144f60b1" />


## Cleanup 

```
kubectl delete -f .
```



## Reference 

  * https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/
