# Install istioctl 

  * Good tool for debugging
  * Download includes demo and additional tools  

### Schritt 1: istio runterladen und installieren 

```
cd 
# current version of istio is 1.29.1
curl -L https://istio.io/downloadIstio | sh -
ln -s ~/istio-1.29.1 ~/istio
echo 'export PATH=~/istio-1.28.0/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Schritt 2: bash completion integrieren 

```
cp ~/istio/tools/istioctl.bash ~/istioctl.bash
echo "source ~/istioctl.bash" >> ~/.bashrc
source ~/istioctl.bash
```

