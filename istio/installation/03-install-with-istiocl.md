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
curl -L https://istio.io/downloadIstio | sh -
mv istio-* istio
echo "export PATH=~/istio/bin:$PATH" >> ~/.bashrc
source ~/.bashrc 
```




## Reference: Get started 

  * https://istio.io/latest/docs/setup/getting-started/
