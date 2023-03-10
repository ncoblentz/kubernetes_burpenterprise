- [Local Microk8s Burp Enterprise Kubernetes PoC](#local-microk8s-burp-enterprise-kubernetes-poc)
  - [Install](#install)
    - [Cluster Setup](#cluster-setup)
      - [Install Microk8s](#install-microk8s)
      - [Setup NFS](#setup-nfs)
      - [Setup Database](#setup-database)
      - [Setup Burp Enterprise](#setup-burp-enterprise)


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

#### Setup Database


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
[burp-namespace.yaml](burp-namespace.yaml)

```bash
$ microk8s kubectl create -f burp-namespace.yaml
```

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

Visit https://bsee-web-server.test:
 - For the Database config enter:
    - JDBC URL: `jdbc:postgresql://postgres.burp.svc.cluster.local:5432/burp_enterprise`
    - Enterprise Server:
      - Username: `burp_enterprise`
      - Password: `changeme`
    - Enterprise Scanning Resources:
      - Username: `burp_agent`
      - Password: `changeme`
