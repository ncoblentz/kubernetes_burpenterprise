- [Local Microk8s Burp Enterprise Kubernetes PoC](#local-microk8s-burp-enterprise-kubernetes-poc)
  - [Install](#install)
    - [Cluster Setup](#cluster-setup)
      - [Install Microk8s](#install-microk8s)
      - [Setup NFS](#setup-nfs)
      - [Create the Kubernetes Namespace](#create-the-kubernetes-namespace)
      - [Setup the Database](#setup-the-database)
      - [Setup Burp Enterprise](#setup-burp-enterprise)
    - [Use Traefik For Ingress](#use-traefik-for-ingress)
    - [Enable Certificate Manager](#enable-certificate-manager)


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

### Enable Certificate Manager

In this example, I am using a CA certificate I generated in order to demonstrate cert-manager's ability to automatically generate server certificates and keep then up-to-date. This matches the environment I was deploying in, in which the server was internally deployed using an internal trusted CA.

__Create a CA Certificate__

```bash
$ mkdir cert
$ cd cert/
$ openssl genrsa -out "root-ca.key" 4096
$ openssl req -new -key "root-ca.key" -out "root-ca.csr" -sha256 -subj '/CN=Local Test Root CA'
$ nano -l root-ca.cnf
$ openssl x509 -req -days 3650 -in "root-ca.csr" -signkey "root-ca.key" -sha256 -out "root-ca.crt" -extfile "root-ca.cnf" -extensions root_ca
Certificate request self-signature ok
subject=CN = Local Test Root CA
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

__Enable Cert-Manager__

```bash
$ microk8s enable cert-manager
```

__Add the CA to the Cluster__

```bash
$ microk8s kubectl -n burp create secret tls ca-pair --cert=cert/root-ca.crt --key=cert/root-ca.key 
secret/ca-pair created
```

[issuer.yaml](issuer.yaml)
```bash
$ microk8s kubectl -n burp create -f issuer.yaml
```

__Option 1: Apply the Certificate to Nginx Ingress__

[burp-ingress-cert-manager.yaml](burp-ingress-cert-manager.yaml)
```bash
$ microk8s kubectl replace -f burp-ingress-cert-manager.yaml 
ingress.networking.k8s.io/bsee-web-server replaced
microk8s kubectl -n burp describe ingress bsee-web-server
Name:             bsee-web-server
Labels:           <none>
Namespace:        burp
Address:          127.0.0.1
Ingress Class:    public
Default backend:  <default>
TLS:
  bsee-web-certificate-ingress terminates bsee-web-server.test
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  bsee-web-server.test  
                        /   bsee-web-server:8443 (10.1.215.154:8443)
Annotations:            cert-manager.io/cluster-issuer: issuer
                        nginx.ingress.kubernetes.io/backend-protocol: HTTPS
Events:
  Type    Reason             Age               From                       Message
  ----    ------             ----              ----                       -------
  Normal  Sync               95s (x3 over 5d)  nginx-ingress-controller   Scheduled for sync
  Normal  CreateCertificate  95s               cert-manager-ingress-shim  Successfully created Certificate "bsee-web-certificate-ingress"
  Normal  Sync               95s (x3 over 5d)  nginx-ingress-controller   Scheduled for sync
$ microk8s kubectl -n burp describe certificate bsee-web-certificate-ingress 
Name:         bsee-web-certificate-ingress
Namespace:    burp
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2023-03-21T15:21:14Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        f:nextPrivateKeySecretName:
    Manager:      cert-manager-certificates-key-manager
    Operation:    Update
    Subresource:  status
    Time:         2023-03-21T15:21:14Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
          .:
          k:{"type":"Ready"}:
            .:
            f:lastTransitionTime:
            f:message:
            f:observedGeneration:
            f:reason:
            f:status:
            f:type:
    Manager:      cert-manager-certificates-readiness
    Operation:    Update
    Subresource:  status
    Time:         2023-03-21T15:21:14Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        f:conditions:
          k:{"type":"Issuing"}:
            .:
            f:lastTransitionTime:
            f:message:
            f:observedGeneration:
            f:reason:
            f:status:
            f:type:
    Manager:      cert-manager-certificates-trigger
    Operation:    Update
    Subresource:  status
    Time:         2023-03-21T15:21:14Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:ownerReferences:
          .:
          k:{"uid":"c341f396-83da-418c-8ecc-18fa6b81d928"}:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:group:
          f:kind:
          f:name:
        f:secretName:
        f:usages:
    Manager:    cert-manager-ingress-shim
    Operation:  Update
    Time:       2023-03-21T15:21:14Z
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  bsee-web-server
    UID:                   c341f396-83da-418c-8ecc-18fa6b81d928
  Resource Version:        1400223
  UID:                     414670fa-860c-4a75-87d8-d7e5147da9a7
Spec:
  Dns Names:
    bsee-web-server.test
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       issuer
  Secret Name:  bsee-web-certificate-ingress
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:        2023-03-21T15:21:14Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      False
    Type:                        Ready
    Last Transition Time:        2023-03-21T15:21:14Z
    Message:                     Issuing certificate as Secret does not exist
    Observed Generation:         1
    Reason:                      DoesNotExist
    Status:                      True
    Type:                        Issuing
  Next Private Key Secret Name:  bsee-web-certificate-ingress-h7svq
Events:
  Type    Reason     Age    From                                       Message
  ----    ------     ----   ----                                       -------
  Normal  Issuing    2m25s  cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  2m25s  cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "bsee-web-certificate-ingress-h7svq"
  Normal  Requested  2m25s  cert-manager-certificates-request-manager  Created new CertificateRequest resource "bsee-web-certificate-ingress-lrw8c"
```

![](2023-03-21-10-27-04.png)

__Option 2: Apply the Certificate to Traefik Ingress Route__

This information was a little difficult for me to find as someone brand-new. This article was really helpful: https://blog.tabbo.it/traefik-secure-ingressroutes/

[burp-certificate.yaml](burp-certificate.yaml)
```bash
$ microk8s kubectl -n burp create -f burp-certificate.yaml 
certificate.cert-manager.io/bsee-web-certificate created
$ microk8s kubectl -n burp describe secrets bsee-web-certificate #it generated a server certificate for us
Name:         bsee-web-certificate
Namespace:    burp
Labels:       <none>
Annotations:  cert-manager.io/alt-names: traefik-ingressroute.bsee-web-server.test,bsee-web-server.test
              cert-manager.io/certificate-name: bsee-web-certificate
              cert-manager.io/common-name: 
              cert-manager.io/ip-sans: 
              cert-manager.io/issuer-group: 
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: issuer
              cert-manager.io/uri-sans: 

Type:  kubernetes.io/tls

Data
====
ca.crt:   1814 bytes
tls.crt:  1525 bytes
tls.key:  1675 bytes
$ microk8s kubectl -n burp describe certificate bsee-web-certificate 
Name:         bsee-web-certificate
Namespace:    burp
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2023-03-21T14:52:46Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:kind:
          f:name:
        f:secretName:
    Manager:      kubectl-create
    Operation:    Update
    Time:         2023-03-21T14:52:46Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        f:revision:
    Manager:      cert-manager-certificates-issuing
    Operation:    Update
    Subresource:  status
    Time:         2023-03-21T14:52:47Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
          .:
          k:{"type":"Ready"}:
            .:
            f:lastTransitionTime:
            f:message:
            f:observedGeneration:
            f:reason:
            f:status:
            f:type:
        f:notAfter:
        f:notBefore:
        f:renewalTime:
    Manager:         cert-manager-certificates-readiness
    Operation:       Update
    Subresource:     status
    Time:            2023-03-21T14:52:47Z
  Resource Version:  1397029
  UID:               f5a0ca04-0963-4e80-b690-50bb3f8140b4
Spec:
  Dns Names:
    traefik-ingressroute.bsee-web-server.test
    bsee-web-server.test
  Issuer Ref:
    Kind:       Issuer
    Name:       issuer
  Secret Name:  bsee-web-certificate
Status:
  Conditions:
    Last Transition Time:  2023-03-21T14:52:47Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2023-06-19T14:52:47Z
  Not Before:              2023-03-21T14:52:47Z
  Renewal Time:            2023-05-20T14:52:47Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    116s  cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  115s  cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "bsee-web-certificate-d7t9w"
  Normal  Requested  115s  cert-manager-certificates-request-manager  Created new CertificateRequest resource "bsee-web-certificate-7vqqf"
  Normal  Issuing    115s  cert-manager-certificates-issuing          The certificate has been successfully issued
```

[burp-ingressroute-cert-manager.yaml ](burp-ingressroute-cert-manager.yaml )

```bash
$ microk8s kubectl replace -f burp-ingressroute-cert-manager.yaml 
ingressroute.traefik.containo.us/burp-ingressroute replaced
```

![](2023-03-21-10-04-24.png)
