# Retries 

```bash
mkdir -p ~/manifests/retry
cd ~/manifests/retry
```

```
nano ratings-retry.yaml
```

```
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
  namespace: bookinfo
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: "5xx"
```

```
kubectl apply -f ratings-retry.yaml
```

```bash
# Watch ratings logs in another terminal
RATINGS_POD=$(kubectl get pod -n bookinfo -l app=ratings -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n bookinfo "$RATINGS_POD" -c ratings -f
```

 * Pod 2 für reviews starten 

```bash
REVIEWS_POD=$(kubectl get pod -n bookinfo -l app=reviews -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n bookinfo -c reviews "$REVIEWS_POD" -- sh -c '
  for i in $(seq 1 10); do
    echo "==== Request $i ===="
    curl -sS http://ratings:9080/ratings/1 || echo "curl failed"
    echo
    sleep 1
  done
'
```

