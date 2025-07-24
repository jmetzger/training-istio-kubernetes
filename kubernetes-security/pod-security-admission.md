# Pod Security Admission 


## Seit: 1.2.22 Pod Security Admission 

  * 1.2.22 - Alpha Feature, was not activated by default. need to activate it as feature gate (Kind)
  * 1.2.23 - Beta -> probably   

## Predefined settings

  * privileges - no restrictions  
  * baseline - some restriction 
  * restricted - really restrictive 

  * Reference: https://kubernetes.io/docs/concepts/security/pod-security-standards/

##  Practical Example starting from Kubernetes 1.23  

```
mkdir -p manifests
cd manifests
mkdir psa 
cd psa 
nano 01-ns.yml 
```

```
# Step 1: create namespace  


apiVersion: v1
kind: Namespace
metadata:
  name: test-ns1
  labels:
    # soft version - running but showing complaints 
    # pod-security.kubernetes.io/enforce: baseline 
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

```

```
kubectl apply -f 01-ns.yml 
```

```
# Schritt 2: Testen mit nginx - pod 
# vi 02-nginx.yml 

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test-ns1
spec:
  containers:
    - image: nginx
      name: nginx
      ports:
        - containerPort: 80

```

```
# a lot of warnings will come up 
# because this image runs as root !! (by default) 
kubectl apply -f 02-nginx.yml
```

```
# Schritt 3:
# Change SecurityContext in  Container 

# vi 02-nginx.yml 

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test-ns1
spec:
  containers:
    - image: nginx
      name: nginx
      ports:
        - containerPort: 80
      securityContext:     
        seccompProfile:    
          type: RuntimeDefault
```

```
kubectl apply -f 02-nginx.yml
```

```
# Schritt 4: 
# Weitere Anpassung runAsNotRoot 
# vi 02-nginx.yml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test-ns<tln>
spec:
  containers:
    - image: nginx
      name: nginx
      ports:
        - containerPort: 80
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        runAsNonRoot: true

```

```
# pod kann erstellt werden, wird aber nicht gestartet 
kubectl apply -f 02-nginx.yml 
```

```
# Schritt 4:
# Anpassen der Sicherheitseinstellung (Phase1) im Container 
```


```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test-ns1
spec:
  containers:
    - image: nginx
      name: nginx
      ports:
        - containerPort: 80
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]

```

```
kubectl delete -f 02-nginx.yml
kubectl apply -f 02-nginx.yml
# von api-server angenommen, ausgerollt, aber kann nicht gestartet werden
kubectl -n test-ns1 get pods
kbuectl -n test-ns1 describe pods 
```

<img width="1454" height="106" alt="image" src="https://github.com/user-attachments/assets/a3565e6c-04da-413c-942c-3a325394500b" />

## Praktisches Beispiel für Version ab 1.2.23 -Lösung - Container als NICHT-Root laufen lassen

  * Wir müssen ein image, dass auch als NICHT-Root laufen kann 
  * .. oder selbst eines bauen (;o)) 
  o bei nginx ist das bitnami/nginx 
 
```
# vi 03-nginx-bitnami.yml 
apiVersion: v1
kind: Pod
metadata:
  name: bitnami-nginx
  namespace: test-ns1
spec:
  containers:
    - image: bitnami/nginx
      name: bitnami-nginx
      ports:
        - containerPort: 80
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        runAsNonRoot: true
```

```
# und er läuft als nicht root 
kubectl apply -f 03_pod-bitnami.yml 
kubectl -n test-ns1 get pods
```
