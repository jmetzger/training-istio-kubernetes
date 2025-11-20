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





## Reference: 

  * https://istio.io/latest/docs/tasks/observability/metrics/querying-metrics/
