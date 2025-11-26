# Deny all AND then only specific stuff (in bookinfo) 

## Graph 

<img width="785" height="470" alt="image" src="https://github.com/user-attachments/assets/bd677a9f-59bd-4aaa-99ca-0dd52f98fda4" />

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


### Test in browser -> url (now productpage should work) 

```
# Wenn das nicht funktioniert SHIFT + Relaod 
im browser: http://<ip>/productpage
```

```
# oder
curl http://<ip>/productpage 
```

## Step 3: Zugriff zu reviews von productpage erlauben 

```
nano 03-reviews-from-productpage.yaml
```

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage-to-reviews
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-productpage"
```

```
kubectl apply -f 03-reviews-from-productpage.yaml
```
### Test in browser -> url (now productpage + reviews should work) 

```
# Wenn das nicht funktioniert SHIFT + Relaod 
im browser: http://<ip>/productpage
```

```
# oder
curl http://<ip>/productpage 
```

## Step 4: Zugriff auf details von productpage 

```
nano 04-details-from-productpage.yaml
```

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage-to-details
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-productpage"
```

```
kubectl apply -f 04-details-from-productpage.yaml
```

### Test in browser -> url (now productpage + reviews + details should work) 

```
# Wenn das nicht funktioniert SHIFT + Relaod 
im browser: http://<ip>/productpage
```

```
# oder
curl http://<ip>/productpage 
```

## Step 5: Zugriff auf ratings von reviews 

```
nano 05-ratings-from-reviews.yaml 
```

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-reviews-to-ratings
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-reviews"
```

```
kubectl apply -f 05-ratings-from-reviews.yaml
```

### Test in browser -> url (now productpage + reviews + details should work) 

```
# Wenn das nicht funktioniert SHIFT + Relaod 
im browser: http://<ip>/productpage
```

```
# oder
curl http://<ip>/productpage 
```
