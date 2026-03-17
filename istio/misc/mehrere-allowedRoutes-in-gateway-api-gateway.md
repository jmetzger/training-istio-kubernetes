# Mehrere allowedRoutes in Gateway Objekt (Gateway API) 

** Mit Label-Selector (empfohlen)**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
```

Dann die gewünschten Namespaces labeln:

```bash
kubectl label ns app1 gateway-access=true
kubectl label ns app2 gateway-access=true
```

```

Es gibt **keine** Möglichkeit, einzelne Namespaces als Liste anzugeben — die Gateway API bietet nur `Same`, `All` oder `Selector`. Der Label-Selector ist der Weg, wenn du gezielt bestimmte Namespaces freischalten willst.
