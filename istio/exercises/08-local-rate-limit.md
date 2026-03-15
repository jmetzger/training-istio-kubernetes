# Local Rate Limit 

## Walkthrough 

  * Bind EnvoyFilter to workload app=productpage

```
cd
mkdir -p manifests/local-rate-limit
cd manifests/local-rate-limit
# Get gateway ip 
kubectl -n bookinfo get gateway bookinfo-gateway
GATEWAY_URL="http://<your-ip-of-gateway>"
```

```
nano rate-limit-240-60s-ok.yaml
```

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-local-ratelimit-svc
  namespace: bookinfo
spec:
  workloadSelector:
    labels:
      app: productpage
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 240
                tokens_per_fill: 240
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              response_headers_to_add:
                - append: false
                  header:
                    key: x-local-rate-limit
                    value: 'true'
```

```
kubectl apply -f rate-limit-240-60s-ok.yaml
```

```
while true; do curl -s "$GATEWAY_URL/productpage" -o /dev/null -w "%{http_code}\n"; done
```

```
# works will out put
```

<img width="87" height="411" alt="image" src="https://github.com/user-attachments/assets/adaa0259-730d-4ba7-aa8d-4e17be50c92b" />

```
# now try with 20 per 60s
```

```
nano rate-limit-20-60s-not.yaml
```

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-local-ratelimit-svc
  namespace: bookinfo
spec:
  workloadSelector:
    labels:
      app: productpage
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 20
                tokens_per_fill: 20
                fill_interval: 60s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              response_headers_to_add:
                - append: false
                  header:
                    key: x-local-rate-limit
                    value: 'true'
```

```
kubectl apply -f rate-limit-20-60s-not.yaml
```

```
while true; do curl -s "$GATEWAY_URL/productpage" -o /dev/null -w "%{http_code}\n"; done
```

```
# now we have a lot of too many connections
# rate limit helps to throttle 
```
<img width="88" height="728" alt="image" src="https://github.com/user-attachments/assets/b70f8898-3a48-4569-8fe2-d998cdc50022" />


