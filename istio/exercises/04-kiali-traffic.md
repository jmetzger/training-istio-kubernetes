# Kiali Traffic

## Start Traffic on commandline 

```
# Set your ip here, you will get it from
# Look for the line with external ip 
kubectl -n bookinfo get svc 
GATEWAY_URL=164.90.237.35 

# This consistantly shows 200, without much a-do 
watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage

```

## Open kiali, and see how it goes 

```
in browser:
# Replace xx with your Teilnehmer-Nr. 
http://kiali.tlnxx.do.t3isp.de

# click on Traffic Graph and choose namespace bookinfo
```

## Mesh 

  * Click on mesh to see if everything is functional


## 
