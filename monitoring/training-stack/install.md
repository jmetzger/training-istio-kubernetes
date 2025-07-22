# Self-Service Stack ausrollen (pro Teilnehmer) 

  * ausgerollt mit terraform (binary ist installiert) - snap install --classic terraform 
  * beinhaltet
      1. 1 controlplane
      1. 3 worker nodes
      1. metallb mit ip's (IP-Adressen) der Nodes (hacky but works)
      1. ingress mit wildcard-domain:  *.tlnx.do.t3isp.de
   
## Walktrough 

  * Setup takes about 6-7 minutes 

```
cd
git clone https://github.com/jmetzger/training-kubernets-monitoring-stack-do-terraform.git install
cd install
cat /tmp/.env
source /tmp/.env
terraform init
terraform apply -auto-approve
```

## Hinweis

```
# Sollte es nicht sauber durchlaufen
# einfach nochmal
terraform apply -auto-approve

# Wenn das nicht geht, einfach nochmal neu
terraform destroy -auto-approve
terraform apply -auto-approve
```

## Testing for ingress-nginx 

  * Let us find out, if svc for nginx is available

```
kubectl -n ingress-nginx get svc
# use this url to access it through curl you should get 404
# e.g.
curl 46.101.239.161
```
