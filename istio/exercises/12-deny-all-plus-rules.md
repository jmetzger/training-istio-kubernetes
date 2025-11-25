# Deny all AND then only specific stuff (in bookinfo) 

## Step 0: Prep:

```
mkdir -p manifests/restrict-access
cd manifests/restrict-access
```

## Step 1: Disallow everything in this namespace 

```
nano 01-restrict-access.yaml
```

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: bookinfo
spec: {}
```

```
kubectl apply -f 01-restrict-access.yaml 
```

### Test in browser url 

```
http://<ip>/productpage
```

## Step 2: Allow access from istio-gateway by namespace 

```
nano 02-from-istio-gateway.yaml
```

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-to-productpage
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        namespaces: ["istio-system"]
```

```
kubectl apply -f 02-from-istio-gateway.yaml
```


## Test in browser -> url (now productpage should work) 


