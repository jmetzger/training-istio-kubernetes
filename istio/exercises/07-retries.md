# Retries 

## Wofür

  * Wenn ein Pod nicht erreicht werden, und einen Code anderen Code als 200 zurückkommt, dann nochmal probieren 

## Vorbereitung: Test-Server erstellen, der 500er erstellt 

```
cd
mkdir -p manifests/retry-flaky
cd manifests/retry-flaky
```

```
nano flaky-server.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: flaky-server
  namespace: bookinfo
data:
  server.py: |
    from http.server import BaseHTTPRequestHandler, HTTPServer
    import random

    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            # 50% Chance auf 500, sonst 200
            if random.random() < 0.5:
                self.send_response(500)
                self.end_headers()
                self.wfile.write(b"oops, random 500\n")
            else:
                self.send_response(200)
                self.end_headers()
                self.wfile.write(b"all good, 200\n")

        def log_message(self, format, *args):
            # Einfaches Logging auf STDOUT
            print("%s - - [%s] %s" % (self.client_address[0], self.log_date_time_string(), format % args))

    if __name__ == "__main__":
        port = 8080
        server = HTTPServer(("", port), Handler)
        print(f"Starting flaky server on port {port}")
        server.serve_forever()

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flaky-server
  template:
    metadata:
      labels:
        app: flaky-server
    spec:
      containers:
      - name: flaky-server
        image: python:3.11-alpine
        command: ["python", "/app/server.py"]
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: server-code
          mountPath: /app
      volumes:
      - name: server-code
        configMap:
          name: flaky-server
          items:
          - key: server.py
            path: server.py
---
apiVersion: v1
kind: Service
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  selector:
    app: flaky-server
  ports:
  - name: http
    port: 8080
    targetPort: 8080
```

```
kubectl apply -f flaky-server.yaml
```


## Schritt 2: VirtualHost ohne retries

```
nano virtual-no-retries.yaml
```

```
# Istio VirtualService mit Retries für flaky-server
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  hosts:
  - flaky-server
  http:
  - route:
    - destination:
        host: flaky-server
        port:
          number: 8080
```

```
kubectl apply -f virtual-no-retries.yaml
```


### Schritt 3: Testscript flaky erstellt  und ausführen 

```
touch flaky-test.sh
chmod u+x flaky-test.sh
nano flaky-test.sh
```

```
#!/usr/bin/env bash

set -e

NAMESPACE="bookinfo"
SERVICE="flaky-server"
PORT=8080

REVIEWS_POD=$(kubectl get pod -n "$NAMESPACE" -l app=reviews -o jsonpath='{.items[0].metadata.name}')
echo "Using REVIEWS_POD: $REVIEWS_POD"
echo "Calling http://$SERVICE:$PORT/ ..."
echo "Zum Abbrechen: STRG+C"
echo

kubectl exec -n "$NAMESPACE" -c reviews "$REVIEWS_POD" -- sh -c '
  i=0
  while true; do
    i=$((i+1))
    code=$(curl -s -o /dev/null -w "%{http_code} in %{time_total}s" http://flaky-server:8080/ || echo "000 in 0")
    echo "[Request $i] -> $code"
    sleep 1
  done
```

```
# 500er beobachte
# Wieviele 500er 
./test-flaky.sh
```

```
CTRL + C 

```

## Schritt 4: virtualservice zu mit retries modifizieren 

```
nano virtualservice-mit-retries.yaml
```

```
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: flaky-server
  namespace: bookinfo
spec:
  hosts:
  - flaky-server
  http:
  - route:
    - destination:
        host: flaky-server
        port:
          number: 8080
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: "5xx,connect-failure,refused-stream,reset"
```

```
kubectl apply -f virtualservice-mit-retries.yaml 
```

```
# nochmal testen
# Wieviele 500er jetzt
./test-flaky.sh
```

```
CTRL + C
```

## Aufräumen 

```
kubectl delete -f .
```
