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

## Beispielwerte (eigene Messung vom 2026-10-03)

Zur Einordnung der Größenordnung. Deine Werte hängen von Hardware und Kernel ab, miss bei dir nach.

  * Umgebung: DigitalOcean, 2 Droplets `s-2vcpu-4gb` (fra1, gleiches Subnetz), kubeadm, Kubernetes 1.37.1, Calico 3.31.2, Ubuntu 24.04, Kernel 6.8
  * Messung: `iperf3` von Pod zu Pod über die Node-Grenze, 10 s je Lauf, 5 Läufe je Variante (Mittelwert)

| Messung | ohne WireGuard | mit WireGuard | Anteil |
|---|---|---|---|
| Pod zu Pod, 1 Stream | 1,86 Gbit/s | 0,74 Gbit/s | 40 % |
| Pod zu Pod, 4 Streams | 1,86 Gbit/s | 0,84 Gbit/s | 45 % |
| Node zu Node (`hostNetwork`) | 1,89 Gbit/s | 1,89 Gbit/s | 100 % |

  * Mit WireGuard bleibt knapp die Hälfte des Durchsatzes übrig.
  * Node-zu-Node-Traffic (Host-Netzwerk) bleibt unverändert: Calico verschlüsselt ihn nicht.
  * Auf den Nodes ist `wg` nicht installiert, `wg show` geht also nicht. Prüfen mit `ip addr list wireguard.cali` (siehe oben) oder am Zähler `/sys/class/net/wireguard.cali/statistics/rx_bytes` auf dem Node.
  * Ausführlich im Artikel: https://jochenimblaumann.de/strategie/pod-traffic-zwischen-nodes-klartext/

## Reference 

  * https://www.tigera.io/blog/introducing-wireguard-encryption-with-calico/
