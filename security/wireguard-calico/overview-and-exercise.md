# Enable wireguard in calico 

## Prerequisites: 

  * Install calicoctl as binary on client 
  * curl -L https://github.com/projectcalico/calico/releases/download/v3.30.2/calicoctl-linux-amd64 -o calicoctl

## Generally speaking: 

  * Calico offers an easy way to enable Wireguard within your cluster
  * This comes with a performance penalty

## Wireguard und Calido 

  * Calico installiert NICHT die wireguard-tools auf dem Server (diese wäre dann mit **wg show** nutzbar) 
  * .. sondern lädt nur das Kernel-Modul und macht mit den Calico-bordmitteln den Rest. 

## Walktrough 

```
# felixconfiguration ist nicht namespace-fähig / also global 
calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
calicoctl get node -o yaml | grep -A 4 -B 4 wireguard
kubectl debug -it node/k8s-w1 --image=busybox -- ip addr list wireguard.cali
```

## Performance Test 

```
# mit wireguard
git clone http://github.com/Pharb/kubernetes-iperf3
cd kubernetes-iperf3
./iperf3.sh
```

```
# Wireguard abschalten
calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":false}}'
```

```
# ohne wireguard
./iperf3.sh
```

## Reference 

  * https://www.tigera.io/blog/introducing-wireguard-encryption-with-calico/
