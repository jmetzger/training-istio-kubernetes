# Exercise networkpolicy calico

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

## Step 2: Namespace nptest ausrollen aus manifests/04-service und testen

```
kubectl create ns nptest
kubectl config set-context --current --namespace=nptest
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

## Step 3: Namespace fremd ausrollen aus manifests/04-service und testen

```
kubectl create ns fremd
```

```
# gleiche Applikation im anderen namespace ausrollen
kubectl -n fremd apply -f .
```

## Step 4: Testen im eigenen Namespace "nptest" und zum -> anderen


```
kubectl run -it --rm access --image=busybox 
```

**Testen zum svc-nginx-Dienst im eigenen Namespace und google**

```
# In der Busybox
# Testinnerhalb meines namespaces 
# bekomme die Ausgabe des Ziels nicht 
wget -O - http://svc-nginx 
# Google geht auch nicht
wget -O - http://www.google.de
# aber dns lookup geht
nslookup www.google.de
```

**Test zum svc-nginx.fremd - Dienst - also im anderen Namespace**

```
# test mit pod im **fremd** namespace
wget -O - http://svc-nginx.fremd 
# --> geht auch nicht 
```

## Step 5: Traffic erlauben egress von busybox 

```
# vi 02-egress-allow-busybox.yml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-busybox-egress
spec:
  order: 10
  selector: run == 'access'
  types:
  - Egress
  egress:
  - action: Allow
```

```
kubectl apply -f 02-egress-allow-busybox.yml
# cnp = calico network policy
kubectl get all,cnp
```

```
kubectl run -it --rm access --image=busybox
```

```
# sollte gehen 
wget -O - http://www.google.de

# sollte nicht funktionieren
wget -O - http://svc-nginx

# sollte nicht funktionieren
# Ausgehender Traffic geht zwar, aber eingehender
# Traffic ist nicht gesetzt (auch nicht im anderen namespace
# Hier haben wir bisher noch keine Regeln
wget -O - http://svc-nginx.fremd 


```

## Step 6: Traffic erlauben für nginx 

```
# 03-allow-ingress-my-nginx.yml 
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-nginx-ingress
spec:
  order: 20 
# für welche pods soll das gelten 
  selector:  web == 'my-nginx'
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
# In der Busybox das geht ->
wget -O - http://svc-nginx
# das nicht
wget -O - http://svc-nginx.fremd
# das geht
wget -O - http://www.google.de
```

## Step 7: Optional: Traffic innerhalb des Namespaces erlauben

```
# alte Regeln rausnehmen
kubectl delete -f 02-egress-allow-busybox.yml
kubectl delete -f 03-allow-ingress-my-nginx.yml
```

```
nano 04-traffic-allowed-inside-namespace.yml
```

```
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-intra-namespace
spec:
  order: 100
  selector: all()
  types:
    - Ingress
    - Egress
  ingress:
    - action: Allow
      source:
        selector: all()
  egress:
    - action: Allow
      destination:
        selector: all()
```

```
kubectl apply -f .
```

```
kubectl run -it --rm access --image=busybox 
```

```
# In der Busybox das geht ->
wget -O - http://svc-nginx
# das nicht
wget -O - http://svc-nginx.fremd
# das geht
wget -O - http://www.google.de
```

## Step 8 (Optional): Traffic vom Ingress Controller erlauben

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
  order: 30
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

