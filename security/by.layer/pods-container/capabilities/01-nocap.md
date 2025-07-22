# Start pod without capabilities 

## Exercise 1:

```
cd
mkdir -p manifests
cd manifests
mkdir -p nocap
cd nocap
```

```
nano 01-pod.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: nocap-nginx 
spec:
  containers:
    - name: web
      image: bitnami/nginx 
      securityContext:
        capabilities:
          drop:
          - all
```


```
kubectl apply -f .
kubectl get pods
kubectl logs nocap-nginx
```


## Exercise 2 

```
nano 02-alpine.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: nocap-alpine 
spec:
  containers:
    - name: web
      command:
        - sleep
        - infinity 
      image: alpine 
      securityContext:
        capabilities:
          drop:
          - all
```


```
kubectl apply -f .
kubectl get pods
kubectl logs nocap-alpine
kubectl exec -it nocap-alpine -- sh 
```

```
ping www.google.de
wget -O - http://www.google.de
```

## Lösung 2: Weitere Capabilities geben 

```
nano 02-alpine.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: nocap-alpine 
spec:
  containers:
    - name: web
      command:
        - sleep
        - infinity 
      image: alpine 
      securityContext:
        capabilities:
          drop:
          - all
# hinzufügen
          add:
          - NET_RAW
```


```
kubectl delete -f 02-alpine.yaml 
kubectl apply -f .
kubectl get pods
kubectl logs nocap-alpine
kubectl exec -it nocap-alpine -- sh 
```

```
ping www.google.de
wget -O - http://www.google.de
```

