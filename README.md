- [Local Microk8s Burp Enterprise Kubernetes PoC](#local-microk8s-burp-enterprise-kubernetes-poc)
  - [Install](#install)
    - [Cluster Setup](#cluster-setup)
      - [Install Microk8s](#install-microk8s)
      - [Setup NFS](#setup-nfs)
      - [Create the Kubernetes Namespace](#create-the-kubernetes-namespace)
      - [Setup the Database](#setup-the-database)
      - [Setup Burp Enterprise](#setup-burp-enterprise)
    - [Use Traefik For Ingress](#use-traefik-for-ingress)


# Local Microk8s Burp Enterprise Kubernetes PoC 

## Install

### Cluster Setup

#### Install Microk8s

```bash
$ sudo apt update && sudo apt upgrade -y && sudo apt install docker
$ sudo snap install microk8s --classic
$ sudo usermod -a -G microk8s $USER
$ sudo chown -f -R $USER ~/.kube
$ newgrp microk8s
$ microk8s status --wait-ready
$ microk8s kubectl get nodes
$ microk8s enable dns helm ingress
```

#### Setup NFS

```bash
$ sudo apt-get install nfs-kernel-server
$ sudo mkdir -p /srv/nfs
$ sudo chown nobody:nogroup /srv/nfs
$ sudo chmod 0777 /srv/nfs
$ #echo '/srv/nfs 10.0.0.0/24(rw,sync,no_subtree_check)' | sudo tee /etc/exports
$ #echo '/srv/nfs 10.0.0.0/24(rw,sync,no_subtree_check,root_squash)' | sudo tee /etc/exports
$ echo '/srv/nfs 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)' | sudo tee /etc/exports
$ sudo systemctl restart nfs-kernel-server
$ #had to change permissions to 777 for /srv/nfs in order for pvc in the cluster to have write permissions - not ideal. I'm not sure what I'm doing wrong - this is a todo
$ microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
$ microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
$    --namespace kube-system \
$    --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
$ microk8s kubectl --namespace=kube-system get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch
$ microk8s kubectl wait pod --selector app.kubernetes.io/name=csi-driver-nfs --for condition=ready --namespace kube-system
```

[sc-nfs.yaml](sc-nfs.yaml)

```bash
$ microk8s kubectl apply -f sc-nfs.yaml
$ microk8s kubectl get storageclasses.storage.k8s.io
```

#### Create the Kubernetes Namespace

[burp-namespace.yaml](burp-namespace.yaml)

```bash
$ microk8s kubectl create -f burp-namespace.yaml
```

#### Setup the Database

[postgres-pvc.yaml](postgres-pvc.yaml)

```bash
$ microk8s kubectl apply -f  postgres-pvc.yaml
```

[postgres-secret.yaml](postgres-secret.yaml)

```bash
$ microk8s kubectl apply -f  postgres-secret.yaml
```
[postgres-pod.yaml](postgres-pod.yaml)

```bash
$ microk8s kubectl apply -f  postgres-pod.yaml 
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "CREATE DATABASE burp_enterprise;"
$ #don't use these passwords below
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "CREATE USER burp_enterprise PASSWORD 'changeme'"
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "CREATE USER burp_agent PASSWORD 'changeme'"
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "GRANT ALL ON DATABASE burp_enterprise TO burp_enterprise"
$ #microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "GRANT ALL ON SCHEMA public TO burp_enterprise" # this is necessary in postgres 15+
```

[postgres-service.yaml](postgres-service.yaml)

```bash
$ microk8s kubectl -n burp create -f postgres-service.yaml 
```

#### Setup Burp Enterprise

[burp-pvc.yaml](burp-pvc.yaml)

```bash
$ microk8s kubectl create -f burp-pvc.yaml
```


Download latest helmchart from: https://portswigger.net/burp/releases/

```bash
$ unzip burp_enterprise_helm_chart_v2023_2.zip -d burphelm
$ microk8s helm show all ./burphelm/
$ nano -l burphelm/values.yaml  #change: persistentVolumeClaim: bsee-pvc -> persistentVolumeClaim: burp-pvc  
$ microk8s helm install -n burp bsee ./burphelm/
$ microk8s kubectl -n burp get service
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
bsee-enterprise-server   ClusterIP   10.152.183.208   <none>        8072/TCP,8073/TCP   5m29s
bsee-web-server          ClusterIP   10.152.183.206   <none>        8080/TCP,8443/TCP   5m29s
```


[burp-ingress.yaml](burp-ingress.yaml)
```bash
$ microk8s kubectl -n burp create -f burp-ingress.yaml
$ echo "10.0.0.15 bsee-web-server.test" | sudo tee -a /etc/hosts
```

~~Create Certificate:~~

```bash
$ #mkdir cert
$ #cd cert/
$ #openssl genrsa -out "root-ca.key" 4096
$ #openssl req -new -key "root-ca.key" -out "root-ca.csr" -sha256 -subj '/CN=Local Test Root CA'
$ #nano -l root-ca.cnf
$ #openssl x509 -req -days 3650 -in "root-ca.csr" -signkey "root-ca.key" -sha256 -out "root-ca.crt" -extfile "root-ca.cnf" -extensions root_ca
#Certificate request self-signature ok
#subject=CN = Local Test Root CA
$ #openssl genrsa -out "server.key" 4096
$ #openssl req -new -key "server.key" -out "server.csr" -sha256 -subj '/CN=*.test'
$ #nano -l server.cnf
$ #openssl x509 -req -days 750 -in "server.csr" -sha256 -CA "root-ca.crt" -CAkey "root-ca.key" -CAcreateserial -out "server.crt" -extfile "server.cnf" -extensions server
#Certificate request self-signature ok
#subject=CN = *.test
$ #openssl pkcs12 -export -out certificate.pfx -inkey  -in certificate.crt -certfile more.crt
#root-ca.cnf  root-ca.crt  root-ca.csr  root-ca.key  server.cnf   server.crt   server.csr   server.key   
$ #openssl pkcs12 -export -out server.pfx -inkey server.key -in server.crt 
#Enter Export Password:
#Verifying - Enter Export Password:
$ #openssl pkcs12 -export -out server.p12 -inkey server.key -in server.crt 
$ #microk8s kubectl -n burp create secret generic myca --from-file=cert/root-ca.crt
```

Visit https://bsee-web-server.test:
- ~~For the certificate, upload `server.p12` with the password you chose~~
- For the Database config enter:
  - JDBC URL: `jdbc:postgresql://postgres.burp.svc.cluster.local:5432/burp_enterprise`
  - Enterprise Server:
    - Username: `burp_enterprise`
    - Password: `changeme`
  - Enterprise Scanning Resources:
    - Username: `burp_agent`
    - Password: `changeme`

### Use Traefik For Ingress

```bash
$ #microk8s disable ingress
$ #microk8s helm repo add traefik https://traefik.github.io/charts
$ #microk8s helm repo update
$ #microk8s helm show values traefik/traefik > traefikvalues.yaml
$ #nano -l traefikvalues.yaml
$ #microk8s helm install -n traefik traefik traefik/traefik
$ microk8s enable community
$ microk8s enable traefik metallb:192.168.2.0/24
$ microk8s kubectl -n traefik port-forward $(microk8s kubectl -n traefik get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
# Then visit http://localhost:9000/dashboard/#/
$ microk8s kubectl -n traefik get services traefik -o jsonpath='{.status.loadBalancer.ingress[*].ip}{"\n"}' # get the IP of the loadbalancer through metallb
192.168.2.0
$ echo "192.168.2.0 traefik-ingressroute.bsee-web-server.test" | sudo tee -a /etc/hosts
$ #echo "10.0.0.15 traefik-ingressroute.bsee-web-server.test" | sudo tee -a /etc/hosts
$ #microk8s kubectl -n traefik get services traefik -o jsonpath="{'web interface node port: '}{.spec.ports[?(@.name=='web')].nodePort}{'\n'}{'websecure interface node port: '}{.spec.ports[?(@.name=='websecure')].nodePort}{'\n'}"
#web interface node port: 31750
#websecure interface node port: 31405
```

[mytransport.yaml](mytransport.yaml)

Traefik won't connect to service/pod with an untrusted TLS certificate. The Burp Enterprise Web Server uses TLS by default with a self signed (or uploaded) certificate. The `mytransport.yaml` file disables certificate validation between Traefik and the Pod, but not between the user's browser and Traefik.

```bash
$ microk8s kubectl -n burp create -f mytransport.yaml
```

[burp-ingressroute.yaml](burp-ingressroute.yaml)

```bash
$ microk8s kubectl -n burp create -f burp-ingressroute.yaml 
```

Visit https://traefik-ingressroute.bsee-web-server.test/