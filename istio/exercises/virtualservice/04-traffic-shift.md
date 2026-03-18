# Traffic Shifting

### 0. Vorbereitung

```bash
mkdir -p ~/manifests/traffic-shifting
cd ~/manifests/traffic-shifting 

# Die Destinationen-Versionen anlegen
cp -a ~/istio/samples/bookinfo/networking/destination-rule-all.yaml destination-rule-all.yaml
kubectl -n bookinfo apply -f destination-rule-all.yaml 
```
### 1. 100% Traffic -> reviews.v1 

```
cat ~/istio/samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/networking/virtual-service-all-v1.yaml
```

```
kubectl get vs -n bookinfo reviews -o yaml | head -n 30
```

### 2. Testen 

```
# Seite öffnen
http://<deine-ip>/productpage

# Egal wie oft du die Seite lädst, es bleibt immer v1
```

### 3. 50% (v1) /50% (v3) Traffic

```
cat ~/istio/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

```bash
kubectl -n bookinfo get vs reviews -o yaml 
```

## 4. Testen 


```
# Seite öffnen
http://<deine-ip>/productpage

# Abwechselnd bei mehrmals laden v1 (keine Sterne) und v3 (sterne)
```

## 5. 100% auf v3 

```
cat ~/istio/samples/bookinfo/networking/virtual-service-reviews-v3.yaml
kubectl -n bookinfo apply -f ~/istio/samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```


```bash
$BOOKINFO_URL="deine-ip"
for i in {1..10}; do
  curl -fsS "$BOOKINFO_URL/productpage" \
    | grep -c 'glyphicon-star"' \
    | awk '{print $1" Sterne"}'
done
```

### 6. Aufräumen 

```
kubectl delete -n bookinfo vs reviews
```

## Reference:

 * https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/
