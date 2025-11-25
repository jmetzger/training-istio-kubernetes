# Install demo-app 

## Step 1: Enable namespace to be part of the mesh 

```
kubectl create ns bookinfo
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient

```
