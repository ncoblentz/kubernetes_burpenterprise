sudo apt-get install nfs-kernel-server
sudo mkdir -p /srv/nfs
sudo chown nobody:nogroup /srv/nfs
sudo chmod 0777 /srv/nfs
#echo '/srv/nfs 10.0.0.0/24(rw,sync,no_subtree_check)' | sudo tee /etc/exports
#echo '/srv/nfs 10.0.0.0/24(rw,sync,no_subtree_check,root_squash)' | sudo tee /etc/exports
echo '/srv/nfs 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)' | sudo tee /etc/exports
sudo systemctl restart nfs-kernel-server
#had to change permissions to 777 for /srv/nfs - not ideal. I'm not sure what I'm doing wrong

microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace kube-system \
    --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
microk8s kubectl --namespace=kube-system get pods --selector="app.kubernetes.io/instance=csi-driver-nfs" --watch
microk8s kubectl wait pod --selector app.kubernetes.io/name=csi-driver-nfs --for condition=ready --namespace kube-system
microk8s kubectl apply -f sc-nfs.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.0.0.15
  share: /srv/nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.2
```
microk8s kubectl get storageclasses.storage.k8s.io

#microk8s kubectl apply -f pvc-nfs.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: burp
spec:
  storageClassName: nfs-csi
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
```


k create -f burp-namespace.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: burp
```
#k create -f burppv.yaml
k create -f burp-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: burp-pvc
  namespace: burp
spec:
  storageClassName: nfs-csi
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 1Gi
```
download latest helmchart from: https://portswigger.net/burp/releases/
unzip burp_enterprise_helm_chart_v2023_2.zip -d burphelm
helm show all ./burphelm/
nano -l burphelm/values.yaml  
  persistentVolumeClaim: bsee-pvc -> persistentVolumeClaim: burp-pvc  
helm install -n burp bsee ./burphelm/
microk8s kubectl -n burp get service
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
bsee-enterprise-server   ClusterIP   10.152.183.208   <none>        8072/TCP,8073/TCP   5m29s
bsee-web-server          ClusterIP   10.152.183.206   <none>        8080/TCP,8443/TCP   5m29s

microk8s kubectl apply -f  postgres-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: burp
spec:
  storageClassName: nfs-csi
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 1Gi
```

microk8s kubectl apply -f  postgres-secret.yaml

```
apiVersion: v1
data:
  POSTGRES_PASSWORD: Y2hhbmdldGhpc3ZhbHVl
kind: Secret
metadata:
  creationTimestamp: null
  name: postgres-secret
  namespace: burp

```

microk8s kubectl apply -f  postgres-pod.yaml 
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: postgres
  name: postgres
  namespace: burp
spec:
  containers:
  - image: postgres:15.2
    name: postgres
    ports:
    - containerPort: 5432
    resources: {}
    volumeMounts:
    - name: postgres-vol
      mountPath: /var/lib/postgresql/data
    envFrom:
    - secretRef:
        name: postgres-secret
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
    - name: postgres-vol
      persistentVolumeClaim:
        claimName: postgres-pvc
status: {}
```

```
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "CREATE DATABASE burp_enterprise;"
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "CREATE USER burp_enterprise PASSWORD 'changeme'"
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "CREATE USER burp_agent PASSWORD 'changeme'"
$ microk8s kubectl -n burp exec -it postgres -- psql -U postgres -c "GRANT ALL ON DATABASE burp_enterprise TO burp_enterprise"
```

microk8s kubectl -n burp create -f postgres-service.yaml 

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: postgres
  name: postgres
  namespace: burp
spec:
  ports:
  - name: 5432-5432
    port: 5432
    protocol: TCP
    targetPort: 5432
  selector:
    run: postgres
  type: ClusterIP
status:
  loadBalancer: {}
```

ingress route will need to be created later

visit webserver at https://10.152.183.206:8443
For Database enter:
JDBC URL: `jdbc:postgresql://postgres.burp.svc.cluster.local:5432/burp_enterprise`
Enterprise Server:
    Username: `burp_enterprise`
    Password: `changeme`
Enterprise Scanning Resources:
    Username: `burp_agent`
    Password: `changeme`

