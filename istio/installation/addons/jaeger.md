# Install jaeger inkl. Ingress (Ingress-Controller must be installed) 

## Prerequisites 

  * *.tlnx.do.t3isp.de subdomain is already set up and pointing to your cluster
>[!NOTE]
>Bitte zunächst Prometheus installieren 

## Step 1: Install addon jaeger 

```
cd 
mkdir -p manifests/jaeger
cd manifests/jaeger
cp -a ~/istio/samples/addons/jaeger.yaml .
kubectl apply -f .
kubectl -n istio-system get pods
kubectl -n istio-system get svc
```

## Step 2: Setup basic auth 

```
nano jaeger-basic-auth.yaml
```

```
# Secret for basic auth: user "training" / password "myS3cr3t!"
# Already base64-encoded in htpasswd format.
apiVersion: v1
kind: Secret
metadata:
  name: jaeger-basic-auth
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
  name: jaeger
  namespace: istio-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: jaeger-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - jaeger'
spec:
  ingressClassName: "nginx"
  rules:
    - host: jaeger.tlnXX.do.t3isp.de
      # ⬆️ Each trainee replaces "XX" with their number, e.g. kiali.tln10.do.t3isp.de
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: tracing
                port:
                  number: 80
```

```
kubectl apply -f .
```

```
# Im browser aufrufen und credentials eingeben (s.o.)
http://jaeger.tlnXX.do.t3isp.de
```
