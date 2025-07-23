# Setup Ingress with Letsencrypt and dns 

## Step 1: Install Cert Manager 

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
## Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

## Install the cert-manager helm chart
helm upgrade --install cert-manager --namespace cert-manager --create-namespace --version v1.14.5 jetstack/cert-manager

```

## Step 2: Create token for digitalocean 

```
# You can create your token here
https://cloud.digitalocean.com/account/api/tokens/new
```

```
# BUT: Trainer already saved the token under /tmp/do_dns_token 
# now you need to encode it
cat /tmp/do_dns_token | base64
```

<img width="847" height="242" alt="image" src="https://github.com/user-attachments/assets/6946bf0f-8117-4a70-b10f-fcf9d4a623b4" />


## Step 2.5: Subdomains einrichten in digitalocean - dns 

```
# Already done 
```

## Step 3: Create secret based on that 

```
cd
mkdir -p manifests
cd manifests
mkdir ssl
cd ssl
```

```
vi 01-secret.yml
```

```
apiVersion: v1
kind: Secret
metadata:
  name: digitalocean-dns
data:
  # insert your DO access token here
  access-token: "base64 encoded access-token here"
```

```
# adjust the lines afterwards !!!
# Achtung: Zeilenumbruch rausnehmen !
cat /tmp/do_dns_token | base64 >> 01-secret.yml
```

```
kubectl -n cert-manager apply -f .
```

## Step 4: Create an clusterissuer (works across all namespaces) 

```
vi 02-clusterissuer.yml
```

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-issuer
spec:
  acme:
    email: some@domain.de
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - selector:
        dnsZones:
          - "do.t3isp.de"
      dns01:
        digitalocean:
          tokenSecretRef:
            name: digitalocean-dns
            key: access-token
```

```
kubectl apply -f .
# Funkioniert ? clusterissuer muss auf ready stehen 
kubectl describe clusterissuer 
```

## Step 5: manifests for certificate (wildcard) 


```
nano 03-certificate.yml
```

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: le-crt
spec:
  secretName: tls-secret
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-prod-issuer
  commonName: "*.tln<nr>.do.t3isp.de"
  dnsNames:
    - "*.tln<nr>.do.t3isp.de"
```

```
kubectl apply -f .
```

## Step 6: check if certificate was created 

```
# certificate request was created - is it ready ? 
kubectl get certificaterequest


kubectl get certificates
kubectl describe certificates le-crt 
kubectl get secret tls-secret -o yaml 
```

## Step 7: Integrate Ingress now 

## Step 7.1: Walkthrough 

```
cd 
cd manifests
mkdir abi 
cd abi
```

```
nano apple-deploy.yml 
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apple-app
  labels:
    app: apple
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apple
  template:
    metadata:
      labels:
        app: apple
    spec:
      containers:
        - name: apple-app
          image: hashicorp/http-echo
          args:
            - "-text=apple-<euer-name>"
```

```
nano apple-svc.yaml
```


```
kind: Service
apiVersion: v1
metadata:
  name: apple-service
spec:
  type: ClusterIP
  selector:
    app: apple
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678 # Default port for image
```

```
kubectl apply -f .
```

```
nano banana-deploy.yml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banana-app
  labels:
    app: banana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: banana
  template:
    metadata:
      labels:
        app: banana
    spec:
      containers:
        - name: apple-app
          image: hashicorp/http-echo
          args:
            - "-text=banana-<euer-name>"
```

```
nano banana-svc.yaml
```

```
kind: Service
apiVersion: v1
metadata:
  name: banana-service
spec:
  type: ClusterIP
  selector:
    app: banana
  ports:
    - port: 80
      targetPort: 5678 # Default port for image
```

```
kubectl apply -f .
```

## Step 7.2: Testing connection by podIP and Service 

```
kubectl get svc
kubectl get pods -o wide
kubectl run podtest --rm -it --image busybox
```

```
/ # wget -O - http://<pod-ip>:5678 
/ # wget -O - http://<cluster-ip>
```

## Step 7.3: Walkthrough 

```
nano ingress.yml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-app
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.tln<nr>.do.t3isp.de
      secretName: tls-secret
  rules:
  - host: "app.tln<nr>.do.t3isp.de"
    http:
      paths:
        - path: /apple
          pathType: Prefix
          backend:
            service:
              name: apple-service
              port:
                number: 80
        - path: /banana
          pathType: Prefix
          backend:
            service:
              name: banana-service
              port:
                number: 80
```

```
kubectl apply -f .
kubectl get ingress
kubectl describe ingress ingress-app
```
