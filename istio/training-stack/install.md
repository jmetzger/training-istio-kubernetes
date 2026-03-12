# Self-Service Stack ausrollen (pro Teilnehmer) 

  * ausgerollt mit terraform (binary ist installiert) - snap install --classic terraform 
  * beinhaltet
      1. 1 controlplane
      1. 3 worker nodes
      1. metallb mit ip's (IP-Adressen) der Nodes (hacky but works)
      1. ingress mit wildcard-domain:  *.tlnx.do.t3isp.de

## Vorbereitung seitens des Trainers

```
# /tmp/.env - Datei wurde vom Trainer vorbereitet
# Inhalt / export -> damit Umgebungsvariable 
export TF_VAR_do_token="DAS_TOKEN_FUER_DIGITALOCEAN"
```

```
Folgende Berechtigungen wurden für das Token gesetzt
```

<img width="1536" height="595" alt="image" src="https://github.com/user-attachments/assets/b394d2b6-82c7-4be6-b8d6-51afc9fd1944" />
   
## Walktrough 

  * Setup takes about 6-7 minutes
  * Hinweis: /tmp/.env beinhaltet Digitalocean Access Token der für das einrichten benötigt wird.

```
cd
git clone https://github.com/jmetzger/training-istio-kubernetes-stack-do-terraform.git install
cd install
cat /tmp/.env
source /tmp/.env
tofu init
tofu apply -auto-approve
```
## Hinweis

```
# Sollte es nicht sauber durchlaufen
# Cluster nochmal löschen 
./scripts/safe-destroy.sh
tofu apply -auto-approve
```

## Testing for ingress-nginx 

  * Let us find out, if svc for nginx is available

```
kubectl -n ingress-nginx get svc
# use this url to access it through curl you should get 404
# e.g.
curl 46.101.239.161
```
