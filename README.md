After you install Ubuntu 18.04 

Step 1: Upgrade the kernel
```
$ sudo apt install linux-generic-hwe-18.04
```
Step 2: Verify the kernel
```
$ uname -sr
Linux 5.4.0-150-generic
```
Step 3: Install gtp5g
```
$ sudo apt -y install git gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev
$ git clone https://github.com/free5gc/gtp5g.git && cd gtp5g
$ sudo make install
```
Step 4: Verify gtp5g is running
```
$ lsmod|grep -i gtp
gtp5g                 122880  0
udp_tunnel             16384  3 gtp5g,wireguard,vxlan
```
Step 5: Install Kubernetes (Microk8s)
```
$ sudo snap install microk8s --classic --channel=1.24/stable
```
Step 6: Configure local user & VM to use local `kubectl` instead of using microk8s all the time
```
$ sudo usermod -a -G microk8s $USER
$ sudo chown -f -R $USER ~/.kube
$ microk8s config
$ cd $HOME
$ mkdir .kube
$ cd .kube
$ microk8s config > config
```
Step 7: Install `kubectl` locally
```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
$ chmod +x kubectl
$ sudo mv kubectl /usr/local/bin/
```
Step 8: Install helm
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
$ helm version
version.BuildInfo{Version:"v3.12.3", GitCommit:"3a31588ad33fe3b89af5a2a54ee1d25bfe6eaa5e", GitTreeState:"clean", GoVersion:"go1.20.7"}
```
Step 9: Disable Calico as it needs additional config, we use flannel instead and reboot
```
$ microk8s disable ha-cluster --force
Infer repository core for addon ha-cluster
Reverting to a non-HA setup
Generating new cluster certificates.
Waiting for node to start. .  
Enabling flanneld and etcd
HA disabled
$ sudo reboot
```
Step 10: Setup persistent volume for Mongo-db

Note: the path being set and the hostname as per the node. In this instance it is ` /home/krupal/kubedata` and `ubuntuk8s` respectively.

```
$ pwd
/home/krupal
$ mkdir kubedata
$ cat pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/krupal/kubedata
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ubuntuk8s
$  kubectl apply -f pv.yaml
persistentvolume/example-local-pv created         
```
Step 11: Add the helm repo
```
$ helm repo add towards5gs 'https://raw.githubusercontent.com/Orange-OpenSource/towards5gs-helm/main/repo/'
```
Step 12: Enable the k8s plugins one by one
```
$ microk8s enable dns ingress dashboard storage community helm3
```
Step 13: Enable Multus plugin & verify cluster is up, `ns` is there, pods are running
```
$ microk8s enable multus
$ kubectl get nodes
$ kubectl get ns
$ kubectl get pods -A -o wide
```
Step 14: Deploy free5gc using helm & verify
```
$ helm -n free5gc install free5gc-v1 towards5gs/free5gc
$ kubectl get pods -A -o wide
NAMESPACE     NAME                                                   READY   STATUS    RESTARTS      AGE     IP           NODE        NOMINATED NODE   READINESS GATES
free5gc       free5gc-v1-free5gc-amf-amf-67b9d48f57-8v8m8            1/1     Running   0             91s     10.1.92.16   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-ausf-ausf-5c5564447b-2cbcr          1/1     Running   0             91s     10.1.92.22   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-dbpython-dbpython-58c46fcc9-n9nq2   1/1     Running   0             91s     10.1.92.21   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-nrf-nrf-55b88d86bd-fwzjj            1/1     Running   0             91s     10.1.92.18   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-nssf-nssf-8b9b8c896-m4mf9           1/1     Running   0             91s     10.1.92.23   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-pcf-pcf-57ddb89cf6-kwpqw            1/1     Running   0             91s     10.1.92.26   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-smf-smf-7f9585589d-929zw            1/1     Running   0             91s     10.1.92.17   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-udm-udm-677d6fdc9-57z8c             1/1     Running   0             91s     10.1.92.20   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-udr-udr-699cd6c75f-c95s6            1/1     Running   0             91s     10.1.92.24   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-upf-upf-f5bd9498f-crn75             1/1     Running   0             91s     10.1.92.15   ubuntuk8s   <none>           <none>
free5gc       free5gc-v1-free5gc-webui-webui-7578968c9b-vnzxj        1/1     Running   0             91s     10.1.92.19   ubuntuk8s   <none>           <none>
free5gc       mongodb-0                                              1/1     Running   0             91s     10.1.92.27   ubuntuk8s   <none>           <none>
ingress       nginx-ingress-microk8s-controller-7phlf                1/1     Running   1 (14m ago)   14m     10.1.92.13   ubuntuk8s   <none>           <none>
kube-system   coredns-66bcf65bb8-c5db6                               1/1     Running   1 (14m ago)   16m     10.1.92.14   ubuntuk8s   <none>           <none>
kube-system   dashboard-metrics-scraper-6b6f796c8d-z485v             1/1     Running   1 (14m ago)   14m     10.1.92.9    ubuntuk8s   <none>           <none>
kube-system   hostpath-provisioner-78cb89d65b-scj8c                  1/1     Running   1 (14m ago)   14m     10.1.92.11   ubuntuk8s   <none>           <none>
kube-system   kube-multus-ds-amd64-nknb9                             1/1     Running   0             5m30s   10.0.5.166   ubuntuk8s   <none>           <none>
kube-system   kubernetes-dashboard-765646474b-j9kwk                  1/1     Running   1 (14m ago)   14m     10.1.92.10   ubuntuk8s   <none>           <none>
kube-system   metrics-server-5f8f64cb86-tw2n5                        1/1     Running   1 (14m ago)   14m     10.1.92.12   ubuntuk8s   <none>           <none>

```
Step 15: Validate Free5GC web ui port, and it is exposed on port 5000 as nodeport service.
```
kubectl get svc -n free5gc
NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
free5gc-v1-free5gc-amf-service    ClusterIP   10.152.183.192   <none>        80/TCP           3m5s
free5gc-v1-free5gc-ausf-service   ClusterIP   10.152.183.176   <none>        80/TCP           3m5s
free5gc-v1-free5gc-nssf-service   ClusterIP   10.152.183.174   <none>        80/TCP           3m5s
free5gc-v1-free5gc-pcf-service    ClusterIP   10.152.183.19    <none>        80/TCP           3m5s
free5gc-v1-free5gc-smf-service    ClusterIP   10.152.183.154   <none>        80/TCP           3m5s
free5gc-v1-free5gc-udm-service    ClusterIP   10.152.183.26    <none>        80/TCP           3m5s
free5gc-v1-free5gc-udr-service    ClusterIP   10.152.183.75    <none>        80/TCP           3m5s
mongodb                           ClusterIP   10.152.183.83    <none>        27017/TCP        3m5s
nrf-nnrf                          ClusterIP   10.152.183.198   <none>        8000/TCP         3m5s
webui-service                     NodePort    10.152.183.122   <none>        5000:30500/TCP   3m5s

```
Step 16:  Port forward to access the Web UI NodePort service

`kubectl port-forward --namespace free5gc svc/webui-service 5000:5000`

Step 17: SSH port forwarding
Note: I've setup a VirtualBox port forwarding on the NATNetwork over localport 2266 to access the node via SSH.
![](https://hackmd.io/_uploads/BJ8YwNwA3.png)

```
ssh -L localhost:5000:localhost:5000 krupal@localhost -p 2266
```











