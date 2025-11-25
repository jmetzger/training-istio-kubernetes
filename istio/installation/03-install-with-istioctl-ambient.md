# Install istio with istioctl (ambient) 

>[NOTE:]
>you need to adjust error from calico with 
>kubectl edit felixconfigurations default -o yaml

```
istioctl install --set profile=ambient --skip-confirmation
# In addition you will need the gateway api - crd's
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml
```
