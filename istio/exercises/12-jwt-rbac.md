# JWT-Roken mit RBAC verwenden 

## Step 1: Create http-bin and curl workloads 

```
kubectl create ns foo
kubectl apply -f <(istioctl kube-inject -f ~/istio/samples/httpbin/httpbin.yaml) -n foo
kubectl apply -f <(istioctl kube-inject -f ~/istio/samples/curl/curl.yaml) -n foo
```





## Reference: 

  * https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/
