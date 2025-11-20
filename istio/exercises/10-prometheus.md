# Istio Metriken auswerten in Prometheus

## Walkthrough 

### Commandline: Send Traffic to the mesh 

```
# Change to your ip accordingly 
GATEWAY_URL=164.90.237.35
curl "http://$GATEWAY_URL/productpage"
```

### Search for metrics 

```
# Login to prometheus.tlnxx.do.t3isp.de
# Click on Query
```

<img width="1894" height="79" alt="image" src="https://github.com/user-attachments/assets/6c1dc59a-0d68-4a0c-9081-0f68a692c4bc" />

```
# in enter expression -> enter
istio_requests_total
# THEN -> Click Execute
```

** This will show all the requests to istio **

```
# You can also click on Graph to see the graphical representation
```

<img width="815" height="416" alt="image" src="https://github.com/user-attachments/assets/2cd43b98-4069-4340-8d91-1199c041bb0c" />

### Try other queries 

```
# 1. Only request to the productpage in namespace bookinfo 
istio_requests_total{destination_service="productpage.bookinfo.svc.cluster.local"}
```

   * Hinweis, wenn du mehrere Einträge siehst wie **hier:**

<img width="1789" height="467" alt="image" src="https://github.com/user-attachments/assets/49881c0f-b462-4ff0-98e3-8bebc22a6566" />

   * .. unterscheiden sich diese bspw. an dieser Stelle durch den Response - Code (200, 403 etc.)

```
# 2. Total count of all requests to v3 of the reviews service:
istio_requests_total{destination_service="reviews.bookinfo.svc.cluster.local", destination_version="v3"}
```

```
# 3.  This query returns the current total count of all requests to the v3 of the reviews service.
# Rate of requests over the past 5 minutes to all instances of the productpage service:
# =~ <- **ist ein regulärer Ausdruck** 
rate(istio_requests_total{destination_service=~"productpage.*", response_code="200"}[5m])
```

## Reference: 

  * https://istio.io/latest/docs/tasks/observability/metrics/querying-metrics/
