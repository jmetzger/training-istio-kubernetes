# Chage installation (example egressgateway) with istioctl and istioOperator

  * Always !! mention the profile
  * Overlays work for: Properties and Scalars, but not for lists
  * To avoid problems always ! list the complete config

## Exercise: Enable egress gateway 

### Step 1: Prepare istioOperator - config 

```
cd
mkdir -p manifests/istio 
cd manfifests/istio
nano istioOperator.yaml
```

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false       # keep this, so it does NOT come back
    egressGateways:
    - name: istio-egressgateway
      enabled: true        # turn this on
```


## Step 2: Dry-run first 

```
# See what is done 
istiocl install -f istioOperator.yaml --dry-run 
```

## Step 3: Install (Change now) 

```
istioclt install -f istioOperator.yaml 

```
