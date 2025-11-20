# Visualization with Grafana 

## Prerequisites 

  * bookinfo app is installed
  * Grafana dashboard is setup here: http://grafana.tlnxx.do.t3isp.de
  * demo profile is used

## Walkthrough 

```
# Open dashboard
http://grafana.tlnxx.do.t3isp.de
```

```
Go to dashboards
```

<img width="337" height="196" alt="image" src="https://github.com/user-attachments/assets/3d60b6ad-0992-4aed-8324-c21d1e8556e2" />

```
Select Dashboard from folder istio ->
```

<img width="526" height="122" alt="image" src="https://github.com/user-attachments/assets/7337c5b4-cbcb-4a91-9971-a73f627b13b5" />

```
# Send traffic to the /productpage
# commandline
# Adjust to your IP 
GATEWAY_URL=164.90.237.35
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
```

```
# In the dashboard click reload (right corner) - a couple of times 
```

```
<img width="334" height="288" alt="image" src="https://github.com/user-attachments/assets/6d117df1-679a-46c5-b52a-82afe68ce138" />
```

```
# The traffic volumen should increase
```

<img width="388" height="255" alt="image" src="https://github.com/user-attachments/assets/254cb10a-1412-4e58-a55d-8612ea8decf6" />

## Next stop: Service Dashboard

  * Click on Dashboards -> Istio -> Service Dashboard

## Next stop: Workload Dashboad 

  * Click on Dashboard -> Istio -> Workload Dashboard


