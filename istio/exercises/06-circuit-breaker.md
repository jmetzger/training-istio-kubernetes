# Circuit Breaker

## Vorbereitung 

```
cd
mkdir -p manifests/circuit-breaker
cd manifests/circuit-breaker 

cp -a ~/istio/samples/bookinfo/platform/kube/bookinfo-version.yaml .
kubectl -n bookinfo apply -f . 

```

## Voraussetzungen 

  * /productpage muss so konfiguriert sie, dass sie zu service 3 geht, falls nein

```
nano 01-100_prozent_v3.yaml
```

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
  namespace: bookinfo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v3
      port: 9080
```

```
kubectl apply -f . 
```

## Destination Rule einsetzen 

  * httpRoute unterstützt kein CircuitBreaker
  * Dieser muss mit DestinationRule eingerichtet werden, der Host legt fest, für welchen Service das gilt

```
nano destinationrule-reviews-cb.yaml
```

```
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-circuit-breaker
  namespace: bookinfo
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        # Sehr wenig Verbindungen, damit der Effekt gut sichtbar ist
        maxConnections: 1
      http:
        # Maximal eine ausstehende HTTP-Anfrage pro Verbindung
        http1MaxPendingRequests: 1
        # Nur eine Anfrage pro Verbindung, damit Verbindungen oft neu aufgebaut werden
        maxRequestsPerConnection: 1
    outlierDetection:
      # Schon bei einem 5xx-Fehler wird ein Pod als "fehlerhaft" markiert
      consecutive5xxErrors: 1
      # Wie oft geprüft wird
      interval: 1s
      # Wie lange ein fehlerhafter Pod aus dem Load-Balancing entfernt wird
      baseEjectionTime: 30s
      # Maximaler Prozentsatz der Pods, die "ausgeschlossen" werden können
      maxEjectionPercent: 100
```

```
kubectl apply -f .
```

```
# Last von intern erzeugen:

kubectl -n bookinfo run curl-cb-test \
  --image=curlimages/curl \
  --restart=Never \
  --command -- sh -c 'while true; do \
    date; \
    echo "=> Request to productpage"; \
    curl -s -o /dev/null -w "HTTP %{http_code}\n" http://productpage:9080/productpage || echo "curl failed"; \
    sleep 0.2; \
  done'
```

```
kubectl -n bookinfo get po curl-cb-test
```

```
NAME           READY   STATUS    RESTARTS   AGE
curl-cb-test   2/2     Running   0          30s
```

```
kubectl -n bookinfo logs -f curl-db-test
```
