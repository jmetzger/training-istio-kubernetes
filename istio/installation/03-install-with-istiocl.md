# Install with istioctl 

  * Most simplistic way
  * Doing the right setup is done with profiles
  * Interestingly it uses an compile-in helm chart  (see also: Show what a profile does)

## Hint for production 





## in our case: Including demo (tracing is activated) 

  * Not suitable for production !!

### Walkthrough 



## Show what a profile does 

```
istioctl manifest generate > istio-manifest.yaml
# If not profile is mentioned, it uses the default profile
# it does not use an operator 
cat istio-manifest.yaml | grep -i -A20 "^Kind" | less
# If you want you can apply it like so:
# kubectl apply -f istio-manifest.yaml 

```

## Installation including Demo 

### Schritt 1: istio runterladen und installieren 

```
cd 
# current version of istio is 1.28.0
curl -L https://istio.io/downloadIstio | sh -
echo "export PATH=~/istio-1.28.0/bin:$PATH" >> ~/.bashrc
source ~/.bashrc 
```

### Schritt 2: bash completion integrieren 

```
cp ~/istio-1.28.0/tools/istioctl.bash ~/istioctl.bash 
source ~/istioctl.bash
```

## Reference: Get started 

  * https://istio.io/latest/docs/setup/getting-started/
