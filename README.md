# Deploy HA Kubernetes Cluster on Linux using kubeadm
**Source: http://kubernetes.io/docs/getting-started-guides/kubeadm/**

## System Requirements

- OS: Ubuntu focal 20.04 or Centos 7, 8
- Platform: On-prem/Cloud/Virtualization
- RAM: 4GB Minimum (we recommend at least 8GB)
- CPU: 2 Minimum (we recommend at least 4CPU)
- NIC: 1 NIC for node
- Firewall: Disabled
- SELinux: Disabled
- SWAP : Disabled

![ha-cluster-k8s-load-balancing-services.jpg](ha-cluster-k8s-load-balancing-services.jpg?raw=true "ha-cluster-k8s-load-balancing-services.jpg")

## Step 1 – Set up Linux all nodes k8s

| TASK               | IP ADDRESS           |  HOSTNAME            |
| -------------------| ---------------------| ---------------------|
| `master-1`         | 192.168.2.104        | master1.rke2.com     |  
| `master-2`         | 192.168.2.105        | master2.rke2.com     |
| `master-3`         | 192.168.2.106        | master3.rke2.com     |
| `load balancer`    | 192.168.2.85         | lb.rke2.com          |
| `worker-1`         | 192.168.2.107        | worker1.rke2.com     |
| `worker-2`         | 192.168.2.108        | worker2.rke2.com     |
| `worker-3`         | 192.168.2.109        | worker3.rke2.com     | 


* Disabled SELinux
```
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
```

* Disabled Firewall
```
systemctl disable firewalld >/dev/null 2>&1
systemctl stop firewalld
```

* Disabled swap
```
sed -i '/swap/d' /etc/fstab
swapoff -a
```

* Install and configure containerd
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
sudo yum install containerd -y
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl enable containerd
systemctl restart containerd
```

* Install Kubernetes packages

Install v1.26.0 kubernetes packages

```
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo yum clean all && sudo yum -y makecache

VERSION=1.26.0

yum install -y kubelet-$VERSION-0 kubeadm-$VERSION-0 kubectl-$VERSION-0 --disableexcludes=kubernetes

yum versionlock kubelet-$VERSION-0 kubeadm-$VERSION-0 kubectl-$VERSION-0
```

* Enable Kubelet
```
sudo systemctl enable --now kubelet
```

# Step 2 – Load Balancer Deployment
- Load balancer can be of the following types:
  - Nginx
  - HAProxy
  - Existing Load Balancer

* Use HAProxy
**Link: https://github.com/buichihau/haproxy-k8s.git**

* Config HAproxy
```
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2
    ulimit-n    65536
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 10240

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats auth admin:oEz1EGCq1pkWnLOm

frontend kube-apiserve
    bind *:6443
    option tcplog
    mode tcp
    default_backend apiserver
frontend rke2
    mode tcp
    bind *:9345
    default_backend rke2_server
    
backend apiserver
    mode tcp
    balance roundrobin
    option tcp-check
    server master-1 192.168.2.104:6443 check fall 3 rise 2
    server master-2 192.168.2.105:6443 check fall 3 rise 2
    server master-3 192.168.2.106:6443 check fall 3 rise 2
backend rke2_server
    mode tcp
   	balance roundrobin
	  option tcp-check
    server master-1 192.168.2.104:9345 check fall 3 rise 2
    server master-2 192.168.2.105:9345 check fall 3 rise 2
    server master-3 192.168.2.106:9345 check fall 3 rise 2
```

#  Step 3 – Set up the First Server Node (Master Node)
* Create cluster (on master node)
```
kubeadm init --control-plane-endpoint=192.168.2.85:6443 --upload-certs --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.26.0
```

* Use kubeadm init to bootstrap the cluster
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.2.85:6443 --token r4w09j.1aeqd3cjk0tl5ttf \
        --discovery-token-ca-cert-hash sha256:91160eda2f93b0f01b82218f78e5735890799b6000cd6e71d03a69ee24d091c0 \
        --control-plane --certificate-key 524dc72e322d3e4dfb44060fa4584c94d8a611da29db28280ad1017be7731b52

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.2.85:6443 --token r4w09j.1aeqd3cjk0tl5ttf \
        --discovery-token-ca-cert-hash sha256:91160eda2f93b0f01b82218f78e5735890799b6000cd6e71d03a69ee24d091c0
```


**In case you lost the kubeadm join command. You can just create the new one with**
```
kubeadm token create --print-join-command
```

* Verify the status of the master node
```
kubectl get node -owide
NAME               STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
master1.rke2.com   Ready    control-plane   24h   v1.26.0   192.168.2.104   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
```

* Create the pod network with Calico
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
```

Link: https://kubernetes.io/docs/concepts/cluster-administration/addons/

* Check if all the system pods and calico pods to change to Running
```
kubectl get pods --all-namespaces

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-56dd5794f-9726h   1/1     Running   0          7d
kube-system   calico-node-6slmv                         1/1     Running   0          2m22s
kube-system   calico-node-qltjv                         1/1     Running   0          7d
kube-system   calico-node-r9t6l                         1/1     Running   0          2m6s
kube-system   coredns-787d4945fb-g5r4n                  1/1     Running   0          7d
kube-system   coredns-787d4945fb-njd7s                  1/1     Running   0          7d
kube-system   etcd-master1.rke2.com                     1/1     Running   0          7d
kube-system   kube-apiserver-master1.rke2.com           1/1     Running   0          7d
kube-system   kube-controller-manager-master1.rke2.com  1/1     Running   0          7d
kube-system   kube-proxy-fnzq8                          1/1     Running   0          2m22s
kube-system   kube-proxy-gtdpm                          1/1     Running   0          7d
kube-system   kube-proxy-qhq2w                          1/1     Running   0          2m6s
kube-system   kube-scheduler-master1.rke2.com           1/1     Running   0          7d
```

* Check out the systemd kubelet , it's no longer crashlooping because it has static pods to start
```
sudo systemctl status kubelet.service
```

* Let's check out the static pod manifests on the Control Plane Node
```
ls /etc/kubernetes/manifests
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

* Setup terminal
```
echo 'alias k=kubectl' >> ~/.bashrc
echo 'alias c=clear' >> ~/.bashrc
```

* install kubeclt
```
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl version
```

* install helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

# Step 4 – Set up additional Server Nodes (Master Nodes)

* Join master node
```
kubeadm join 192.168.2.85:6443 --token r4w09j.1aeqd3cjk0tl5ttf \
      --discovery-token-ca-cert-hash sha256:91160eda2f93b0f01b82218f78e5735890799b6000cd6e71d03a69ee24d091c0 \
      --control-plane --certificate-key 524dc72e322d3e4dfb44060fa4584c94d8a611da29db28280ad1017be7731b52
```


* After some time, check the status of the nodes

```
 kubectl get node -owide
NAME                                   STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
master1.rke2.com    Ready    control-plane   36h   v1.26.0   192.168.2.104   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
master2.rke2.com   Ready    control-plane   35h   v1.26.0   192.168.2.105   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
master3.rke2.com    Ready    control-plane   35h   v1.26.0   192.168.2.106   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
```

* Check if all the system pods and calico pods to change to Running
```
kubectl get pods --all-namespaces

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-56dd5794f-9726h   1/1     Running   0          7d
kube-system   calico-node-6slmv                         1/1     Running   0          2m22s
kube-system   calico-node-qltjv                         1/1     Running   0          7d
kube-system   calico-node-r9t6l                         1/1     Running   0          2m6s
kube-system   coredns-787d4945fb-g5r4n                  1/1     Running   0          7d
kube-system   coredns-787d4945fb-njd7s                  1/1     Running   0          7d
kube-system   etcd-master1.rke2.com                     1/1     Running   0          7d
kube-system   etcd-master2.rke2.com                     1/1     Running   0          7d
kube-system   etcd-master3.rke2.com                     1/1     Running   0          7d
kube-system   kube-apiserver-master1.rke2.com           1/1     Running   0          7d
kube-system   kube-apiserver-master2.rke2.com           1/1     Running   0          7d
kube-system   kube-apiserver-master3.rke2.com           1/1     Running   0          7d
kube-system   kube-controller-manager-master1.rke2.com  1/1     Running   0          7d
kube-system   kube-controller-manager-master2.rke2.com  1/1     Running   0          7d
kube-system   kube-controller-manager-master3.rke2.com  1/1     Running   0          7d
kube-system   kube-proxy-fnzq8                          1/1     Running   0          2m22s
kube-system   kube-proxy-gtdpm                          1/1     Running   0          7d
kube-system   kube-proxy-qhq2w                          1/1     Running   0          2m6s
kube-system   kube-scheduler-master1.rke2.com           1/1     Running   0          7d
kube-system   kube-scheduler-master2.rke2.com           1/1     Running   0          7d
kube-system   kube-scheduler-master3.rke2.com           1/1     Running   0          7d
```
# Step 5 – Set up Agent Nodes (Worker Nodes)

* Join any number of worker nodes
```
kubeadm join 192.168.2.85:6443 --token r4w09j.1aeqd3cjk0tl5ttf \
        --discovery-token-ca-cert-hash sha256:91160eda2f93b0f01b82218f78e5735890799b6000cd6e71d03a69ee24d091c0
```

* Verify cluster status (on CP node)
```
kubectl get node -owide
NAME                STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
master1.rke2.com    Ready    control-plane   36h   v1.26.0   192.168.2.104   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
master2.rke2.com    Ready    control-plane   35h   v1.26.0   192.168.2.105   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
master3.rke2.com    Ready    control-plane   35h   v1.26.0   192.168.2.106   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
worker1.rke2.com    Ready    <none>          35h   v1.26.0   192.168.2.107   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
worker2.rke2.com    Ready    <none>          35h   v1.26.0   192.168.2.108   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
worker3.rke2.com    Ready    <none>          35h   v1.26.0   192.168.2.109   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.6.27
```

## 2. Install Metrics Server with helm

* Create file install
```
mkdir -p /k8s
cd /k8s
```

* install Metrics Server
```
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm search repo metrics-server
helm pull metrics-server/metrics-server --version 3.8.2
tar -xzf metrics-server-3.8.2.tgz
```

* fix, edit file value.yaml

vim /k8s/metrics-server/values.yaml

```
defaultArgs:
  - --cert-dir=/tmp
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
  - --secure-port=4443
  - --kubelet-insecure-tls=true
```

* Deploy metric-server
```
cd /k8s/
helm install metric-server metrics-server -n kube-system
```

* Verify metric-server
```
kubectl -n kube-system get pods |grep metric
metric-server-metrics-server-97f9cf9c7-g4c6x                   1/1     Running   0               3m56s
```
## 3. Install NGINX ingress controller

### 3.1 Install NGINX ingress controller by bitnami

**Source: https://artifacthub.io/packages/helm/bitnami/nginx-ingress-controller**

* create namespace nginx-ingress
```
kubectl create ns nginx-ingress
```

* Pull source chart nginx-ingress-controller
```
cd /k8s/
helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/nginx-ingress-controller --untar --version 10.1.0
```

* Change conf service type from LoadBalancer to nodeport and kind from Deployment to DaemonSet

vim /k8s/nginx-ingress-controller/values.yaml

```
kind: DaemonSet
daemonset:
  useHostPort: true

service:
  type: NodePort
  ports:
    http: 80
    https: 443
  targetPorts:
    http: http
    https: https
  nodePorts:
    http: "30100"
    https: "30101"
    tcp: {}
    udp: {}
```

* Deploy nginx-ingress-controller
```
cd /k8s/nginx-ingress-controller
helm install -n nginx-ingress -f values.yaml nginx-ingress-controller .
```

* Verify nginx-ingress-controller
```
kubectl get all -n nginx-ingress
NAME                                                            READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-9p4br                              1/1     Running   0          2m53s
pod/nginx-ingress-controller-default-backend-5b6d74fcfb-d48jn   1/1     Running   0          7h12m
pod/nginx-ingress-controller-jjjts                              1/1     Running   0          7h12m
pod/nginx-ingress-controller-vqq64                              1/1     Running   0          7h12m

NAME                                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress-controller                   NodePort    10.96.158.41    <none>        80:30100/TCP,443:30101/TCP   7h12m
service/nginx-ingress-controller-default-backend   ClusterIP   10.109.47.130   <none>        80/TCP                       7h12m

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/nginx-ingress-controller   3         3         3       3            3           <none>          7h12m

NAME                                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller-default-backend   1/1     1            1           7h12m

NAME                                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-controller-default-backend-5b6d74fcfb   1         1         1       7h12m
```

### 3.2 Install NGINX ingress controller by nginx

**Source: https://kubernetes.github.io/ingress-nginx/deploy/**

* create namespace nginx-ingress
```
kubectl create ns nginx-ingress
```

* Pulling the Chart
```
cd /k8s/
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm pull ingress-nginx/ingress-nginx --untar --version 4.9.0
```

* Change conf service type from LoadBalancer to nodeport and kind from Deployment to DaemonSet

vim /k8s/ingress-nginx/values.yaml

```
service:
  type: NodePort
  nodePorts:
      # -- Node port allocated for the external HTTP listener. If left empty, the service controller allocates one from the configured node port range.
      http: "30100"
      # -- Node port allocated for the external HTTPS listener. If left empty, the service controller allocates one from the configured node port range.
      https: "30101"
```

* Deploy nginx-ingress-controller
```
cd /k8s/ingress-nginx
helm install -n nginx-ingress ingress-nginx .
```

* Verify nginx-ingress-controller

```
kubectl get all -n nginx-ingress
NAME                                 READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-cprzp   1/1     Running   0          91m
pod/ingress-nginx-controller-pgzc9   1/1     Running   0          91m
pod/ingress-nginx-controller-v8bqv   1/1     Running   0          91m

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.96.152.204   <none>        80:30100/TCP,443:30101/TCP   91m
service/ingress-nginx-controller-admission   ClusterIP   10.102.167.58   <none>        443/TCP                      91m

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/ingress-nginx-controller   3         3         3       3            3           kubernetes.io/os=linux   91m

```

## Run End-to-end test

* Deploy nginx 
```
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

* Verify pod can run
```
kubectl get pods
kubectl get deployments
```

* View logs
```
kubectl logs -f pod-id
```

* Service can access
```
kubectl expose deployment nginx-deployment --type NodePort --port 80
kubectl get services
```

* Check access
```
curl --head http://localhost:nodeport
```

## 4. Etcd Backup and Restore on Kubernetes 

![etcd.png](etcd.png?raw=true "etcd.png")

* install etcdctl 
```
sudo apt install etcd-client
```

* Validate info 
etcd endpoint (–endpoints)
ca certificate (–cacert)
server certificate (–cert)
server key (–key)