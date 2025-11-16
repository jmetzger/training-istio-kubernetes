# Install with istioctl 

  * Most simplistic way
  * Doing the right setup is done with profiles
  * Interestingly it uses an compile-in helm chart  (see also: Show what a profile does)

## Hint for production 

  * Bet option is default 



## in our case: Including demo (tracing is activated) 

  * Not suitable for production !!

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
ln -s ~/istio-1.28.0 ~/istio
echo "export PATH=~/istio-1.28.0/bin:$PATH" >> ~/.bashrc
source ~/.bashrc 
```

### Schritt 2: bash completion integrieren 

```
cp ~/istio/tools/istioctl.bash ~/istioctl.bash 
source ~/istioctl.bash
```

### Schritt 3: Installation 

```
# cat ~/istio/samples/bookinfo/demo-profile-no-gateways.yaml
# Wird vom ControlPlane ausgewertet
# Hier wird das ingressgateway abgeschaltet,
# Weil wir das nicht benötigen, wenn wir
# die Kubernetes Gateway API verwenden 
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false
    egressGateways:
    - name: istio-egressgateway
      enabled: false
```


```
# Der Trend geht Richtung Kubernetees Gateway API
istioctl install -f ~/istio/samples/bookinfo/demo-profile-no-gateways.yaml -y
```

## Reference: Get started 

  * https://istio.io/latest/docs/setup/getting-started/
