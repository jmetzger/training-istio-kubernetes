# runAs 

  * Best practices. Set on level of the pod
  * This also reflects containers that are started with kubectl debug (ephemerla containers)


## runAsUser 

  * Important: UID does not need to exist in container 
  * Really run as specific user.
  * If not set the user from Dockerfile is taken
  * Recommended to set it, that will be deep defense line
    * If image has a root user or can not run as root this will fail 

## Exercise 1 

```
cd
mkdir -p manifests
cd manifests
mkdir run1
cd run1
```

```
nano 01-pod.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: nginxrun
spec:
  securityContext:
    runAsUser: 10001
  containers:
  - image: nginx:1.23
    name: pod
  restartPolicy: Always
```

```
kubectl apply -f 01-pod.yaml
kubectl get pods 
```

```
kubectl describe pod nginxrun
kubectl logs nginxrun 
```

## Lösung 1.1 Alternative image verwenden 

```
kubectl delete -f 01-pod.yaml
```

```
# - image: nginx:1.23 -> durch 
- image: bitnami/nginx:1.23 
```

```
kubectl apply -f .
```


## Exercise 2: (works) 

```
cd
mkdir -p manifests
cd manifests
mkdir run2
cd run2
```

```
nano 01-pod.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: alpinerun
spec:
  securityContext:
    runAsUser: 10001
  containers:
  - image: alpine
    command:
      - sleep
      - infinity 
    name: pod
```

```
kubectl apply -f .
kubectl describe pod alpinerun
kubectl exec -it alpinerun -- sh
```

```
id
cd /proc/1/ns
ls -la
```

## runAsGroup 

  * Recommended to set this as well 

## runAsNonRoot 

  * Indicates that use must run as none root
  * If this is not configure in the image, the start fails
