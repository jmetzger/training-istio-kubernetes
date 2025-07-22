# Setup Ingress with Letsencrypt and dns 

## Step 1: Install Cert Manager 

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
## Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

## Install the cert-manager helm chart
helm install cert-manager --namespace cert-manager --create-namespace --version v1.14.5 jetstack/cert-manager

```

## Step 2: Create token for digitalocean 

```
# You can create your token here
https://cloud.digitalocean.com/account/api/tokens/new
```

<img width="847" height="242" alt="image" src="https://github.com/user-attachments/assets/6946bf0f-8117-4a70-b10f-fcf9d4a623b4" />


```
# we save the token under /tmp/do_dns_token 
# now you need to encode it
cat /tmp/do_dns_token | base64
```

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
```

## Step 5: manifests for certificate (wildcard) 

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
kubectl get certificates
kubectl describe certificates let-cert 
kubectl get secret tls-secret -o yaml 
```

