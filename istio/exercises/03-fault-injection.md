# Fault Injection (sidecar-mode) 

  * The gateway api does not support fault-injection (so the http-route object)
  * We are using the gateway api, but can still use the VirtualService for Service-2-Service call inside the mesh

## Walkthrough 

### Step 1: Prepare 

```
cd
mkdir -p manifests/fault-injection
cd manifests/fault-injection
```

```
cp -a ~/istio/samples/bookinfo/networking/virtual-service-all-v1.yaml . 
cp -a ~/istio/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml .
kubectl -n bookinfo apply -f . 
```

### Step 2: Fault-Injection - Abort (500er)

```
cp -a ~/istio/samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml . 
kubectl apply -f virtual-service-ratings-test-abort.yaml 
```


### Step 3: Seite aufrufen und als jason einloggen

  * <dein-ip-des-gateway>/productpage 

<img width="929" height="171" alt="image" src="https://github.com/user-attachments/assets/37b241df-dd02-44c1-bd18-2797b405bac5" />

### Step 4: Ausloggen oder als andere Nutzer einloggen


  * Reviews gehen wieder 
