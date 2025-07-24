# Exercise networkpolicy calico

## Prepare (Step 0: :o)):

  * Use a specific namespace for that

```
kubectl create ns nptest 
kubectl config set-context --current --namespace=nptest 
```


## Step 1: Set global policy

```
cd
mkdir -p manifests/calico
cd manifests/calico
nano 01-gp.yml
```

  * Best practice no "order" here, will be processed last then

```
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  namespaceSelector: has(kubernetes.io/metadata.name) && kubernetes.io/metadata.name not in {"kube-system", "calico-system", "tigera-operator"}
  # namespaceSelector: kubernetes.io/metadata.name != "kube-system" && kubernetes.io/metadata.name != "calico-system" && kubernetes.io/metadata.name != 'tigera-operator'
  types:
  - Ingress
  - Egress
  egress:
   # allow all namespaces to communicate to DNS pods
  - action: Allow
    protocol: UDP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53
  - action: Allow
    protocol: TCP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53

```

```
kubectl apply -f . 
```

## Step 2: nginx ausrollen aus manifests/04-service und testen

```
# pod in anderem namespace starten
kubectl create ns somewhere
kubectl run -n somewhere nginx-other-namespace --image=nginx
# Get ip and note down -> e.g. 192.168.46.20
kubectl -n somewhere get pods -o wide 
```

```
cd
mkdir -p manifests
cd manifests
mkdir 04-service 
cd 04-service 
```

```
nano deploy.yml 
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-nginx
spec:
  selector:
    matchLabels:
      web: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        web: my-nginx
    spec:
      containers:
      - name: cont-nginx
        image: nginx
        ports:
        - containerPort: 80
```

```
nano service.yml
```


```
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  labels:
    run: svc-my-nginx
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
  selector:
    web: my-nginx      
        
```        

```
kubectl apply -f . 
```

```
kubectl run -it --rm access --image=busybox 
```
```
# In der Busybox 
# bekomme die Ausgabe des Ziels nicht 
wget -O - http://svc-nginx 
# Google geht auch nicht
wget -O - http://www.google.de
# aber dns lookup geht
nslookup www.google.de
```

```
# test mit pod im somewhere namespace
# wget -O - http://<ip-deines-somewhere-pods>
# z.B. 
wget -O - 192.168.46.20
# --> geht auch nicht 
```

## Step 3: Traffic erlauben egress von busybox 

  * Achtung: Sei wachsam Holzauge 

```
cd
cd manifests
mkdir cnp
cd cnp
```


```
# vi 02-egress-allow-busybox.yml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-busybox-egress
spec:
  selector: run == 'access'
  types:
  - Egress
  egress:
  - action: Allow
```

```
kubectl apply -f . 
```

```
kubectl run -it --rm access --image=busybox
```

```
# sollte gehen 
wget -O - http://www.google.de

# sollte nicht funktionieren
wget -O - http://my-nginx
```

## Step 4: Traffic erlauben für nginx 

```
# 03-allow-ingress-my-nginx.yml 
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-nginx-ingress
spec:
  selector: run == 'my-nginx'
  types:
  - Ingress
  ingress:
  - action: Allow
    source:
      selector: run == 'access'
```

```
kubectl apply -f .
```

```
kubectl run -it --rm access --image=busybox 
```

```
# In der Bbusybox 
wget -O - http://my-nginx 
```

## Step 5 (Optional): Traffic vom Ingress Controller erlauben

```
nano ingress-network-policy.yaml
```

```
# 04-allow-ingress-controller-to-nginx.yml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller-to-nginx
spec:
  selector: web == 'my-nginx'
  types:
  - Ingress
  ingress:
  - action: Allow
    source:
      namespaceSelector: kubernetes.io/metadata.name == "ingress-nginx"
    destination:
      ports: [80, 443]
```

```
kubectl apply -f .
```
