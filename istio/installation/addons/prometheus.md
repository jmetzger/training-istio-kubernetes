# Install prometheus inkl. Ingress (Ingress-Controller must be installed) 

## Prerequisites 

  * *.tlnx.do.t3isp.de subdomain is already set up and pointing to your cluster
    
## Step 1: Install addon prometheus 

```
cd 
mkdir -p manifests/prometheus
cd manifests/prometheus
cp -a ~/istio/samples/addons/prometheus.yaml .
kubectl apply -f .
kubectl -n istio-system get pods
kubectl -n istio-system get svc
```

## Step 2: Setup basic auth 

```
nano prometheus-basic-auth.yaml
```

```
# Secret for basic auth: user "training" / password "myS3cr3t!"
# Already base64-encoded in htpasswd format.
apiVersion: v1
kind: Secret
metadata:
  name: prometheus-basic-auth
  namespace: istio-system
type: Opaque
data:
  # htpasswd-style content:
  # training:$2b$12$CfOZaJ.Tr0zu6PfpbuCjzeKiQ2PzZARfP.CbC6tRU/7OvEHCIOREm
  auth: dHJhaW5pbmc6JDJiJDEyJENmT1phSi5UcjB6dTZQZnBidUNqemVLaVEyUHpaQVJmUC5DYkM2dFJVLzdPdkVIQ0lPUkVtCg==
```

```
kubectl apply -f .
```

### Step 3: Setup Ingress 

```
nano ingress.yaml
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: prometheus-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - kiali'
spec:
  ingressClassName: "nginx"
  rules:
    - host: prometheus.tlnXX.do.t3isp.de
      # ⬆️ Each trainee replaces "XX" with their number, e.g. prometheus.tln10.do.t3isp.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus
                port:
                  number: 9090
```

```
kubectl apply -f .
```

```
# Im browser aufrufen und credentials eingeben (s.o.)
http://prometheus.tlnXX.do.t3isp.de
```
