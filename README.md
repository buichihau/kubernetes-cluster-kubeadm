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

* RKE2 config file for second server (master2)
```
cat >>/etc/rancher/rke2/config.yaml<<EOF
server: https://192.168.2.85:9345
token: rke2-k8s-sTill-win-@-zay
tls-san:
  - lb.rke2.com
  - 192.168.2.85
  - 192.168.2.104
  - 192.168.2.105
  - 192.168.2.106
  - master1.rke2.com 
  - master2.rke2.com
  - master3.rke2.com 
node-ip: 192.168.2.105
node-name: master2.rke2.com
EOF
```

* RKE2 config file for third  server (master3)
```
cat >>/etc/rancher/rke2/config.yaml<<EOF
server: https://192.168.2.85:9345
token: rke2-k8s-sTill-win-@-zay
tls-san:
  - lb.rke2.com
  - 192.168.2.85
  - 192.168.2.104
  - 192.168.2.105
  - 192.168.2.106
  - master1.rke2.com 
  - master2.rke2.com
  - master3.rke2.com 
node-ip: 192.168.2.106
node-name: master3.rke2.com
EOF
```

* start the service
```
sudo systemctl start rke2-server
sudo systemctl enable rke2-server
```
* After some time, check the status of the nodes
```
kubectl get node -owide
NAME               STATUS   ROLES                       AGE   VERSION           INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
master1.rke2.com   Ready    control-plane,etcd,master   10h   v1.26.12+rke2r1   192.168.2.104   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
master2.rke2.com   Ready    control-plane,etcd,master   22m   v1.26.12+rke2r1   192.168.2.105   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
master3.rke2.com   Ready    control-plane,etcd,master   11m   v1.26.12+rke2r1   192.168.2.106   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
```

# Step 5 – Set up Agent Nodes (Worker Nodes)

* Install RKE2 agent
```
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
sudo INSTALL_RKE2_TYPE=agent ./install.sh
```

* RKE2 config file for Agent Nodes (Worker 1)
```
cat >>/etc/rancher/rke2/config.yaml<<EOF
server: https://192.168.2.85:9345
token: rke2-k8s-sTill-win-@-zay
tls-san:
  - lb.rke2.com
  - 192.168.2.85
  - 192.168.2.104
  - 192.168.2.105
  - 192.168.2.106
  - master1.rke2.com 
  - master2.rke2.com
  - master3.rke2.com 
node-ip: 192.168.2.107
node-name: worker1.rke2.com
EOF
```

* RKE2 config file for Agent Nodes (Worker 2)
```
cat >>/etc/rancher/rke2/config.yaml<<EOF
server: https://192.168.2.85:9345
token: rke2-k8s-sTill-win-@-zay
tls-san:
  - lb.rke2.com
  - 192.168.2.85
  - 192.168.2.104
  - 192.168.2.105
  - 192.168.2.106
  - master1.rke2.com 
  - master2.rke2.com
  - master3.rke2.com 
node-ip: 192.168.2.108
node-name: worker2.rke2.com
EOF
```

* RKE2 config file for Agent Nodes (Worker 3)
```
cat >>/etc/rancher/rke2/config.yaml<<EOF
server: https://192.168.2.85:9345
token: rke2-k8s-sTill-win-@-zay
tls-san:
  - lb.rke2.com
  - 192.168.2.85
  - 192.168.2.104
  - 192.168.2.105
  - 192.168.2.106
  - master1.rke2.com 
  - master2.rke2.com
  - master3.rke2.com 
node-ip: 192.168.2.109
node-name: worker3.rke2.com
EOF
```

* start the service rke2-agent
```
sudo systemctl start rke2-agent
sudo systemctl enable rke2-agent
```

* After some time, check the status of the nodes
```
 kubectl get node -owide
NAME               STATUS   ROLES                       AGE    VERSION           INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
master1.rke2.com   Ready    control-plane,etcd,master   12h    v1.26.12+rke2r1   192.168.2.104   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
master2.rke2.com   Ready    control-plane,etcd,master   130m   v1.26.12+rke2r1   192.168.2.105   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
master3.rke2.com   Ready    control-plane,etcd,master   119m   v1.26.12+rke2r1   192.168.2.106   <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
worker1.rke2.com   Ready    <none>                      23m    v1.26.12+rke2r1   192.168.2.107    <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
worker2.rke2.com   Ready    <none>                      23m    v1.26.12+rke2r1   192.168.2.108    <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
worker3.rke2.com   Ready    <none>                      23m    v1.26.12+rke2r1   192.168.2.109    <none>        CentOS Linux 7 (Core)   3.10.0-1160.90.1.el7.x86_64   containerd://1.7.11-k3s2
```

* Check pods
```
kubectl get pod -A
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE
kube-system   cloud-controller-manager-master1.rke2.com               1/1     Running     0          12h
kube-system   cloud-controller-manager-master2.rke2.com               1/1     Running     0          132m
kube-system   cloud-controller-manager-master3.rke2.com               1/1     Running     0          120m
kube-system   etcd-master1.rke2.com                                   1/1     Running     0          12h
kube-system   etcd-master2.rke2.com                                   1/1     Running     0          131m
kube-system   etcd-master3.rke2.com                                   1/1     Running     0          120m
kube-system   helm-install-rke2-canal-l28sd                           0/1     Completed   0          12h
kube-system   helm-install-rke2-coredns-8pb9f                         0/1     Completed   0          12h
kube-system   helm-install-rke2-ingress-nginx-sdvh6                   0/1     Completed   0          12h
kube-system   helm-install-rke2-metrics-server-9rkhv                  0/1     Completed   0          12h
kube-system   helm-install-rke2-snapshot-controller-7s5xs             0/1     Completed   1          12h
kube-system   helm-install-rke2-snapshot-controller-crd-88rf8         0/1     Completed   0          12h
kube-system   helm-install-rke2-snapshot-validation-webhook-lvhwl     0/1     Completed   0          12h
kube-system   kube-apiserver-master1.rke2.com                         1/1     Running     0          12h
kube-system   kube-apiserver-master2.rke2.com                         1/1     Running     0          132m
kube-system   kube-apiserver-master3.rke2.com                         1/1     Running     0          120m
kube-system   kube-controller-manager-master1.rke2.com                1/1     Running     0          12h
kube-system   kube-controller-manager-master2.rke2.com                1/1     Running     0          132m
kube-system   kube-controller-manager-master3.rke2.com                1/1     Running     0          120m
kube-system   kube-proxy-master1.rke2.com                             1/1     Running     0          12h
kube-system   kube-proxy-master2.rke2.com                             1/1     Running     0          132m
kube-system   kube-proxy-master3.rke2.com                             1/1     Running     0          120m
kube-system   kube-proxy-worker3.rke2.com                             1/1     Running     0          25m
kube-system   kube-proxy-worker2.rke2.com                             1/1     Running     0          25m
kube-system   kube-proxy-worker1.rke2.com                             1/1     Running     0          25m
kube-system   kube-scheduler-master1.rke2.com                         1/1     Running     0          12h
kube-system   kube-scheduler-master2.rke2.com                         1/1     Running     0          132m
kube-system   kube-scheduler-master3.rke2.com                         1/1     Running     0          120m
kube-system   rke2-canal-5l4sl                                        2/2     Running     0          121m
kube-system   rke2-canal-8dqnt                                        2/2     Running     0          12h
kube-system   rke2-canal-hgl8c                                        2/2     Running     0          132m
kube-system   rke2-canal-ddfjc                                        2/2     Running     0          25m
kube-system   rke2-canal-fjfjc                                        2/2     Running     0          25m
kube-system   rke2-canal-mj5jc                                        2/2     Running     0          25m
kube-system   rke2-coredns-rke2-coredns-565dfc7d75-9m7pt              1/1     Running     0          132m
kube-system   rke2-coredns-rke2-coredns-565dfc7d75-vmn8h              1/1     Running     0          12h
kube-system   rke2-coredns-rke2-coredns-autoscaler-6c48c95bf9-852zt   1/1     Running     0          12h
kube-system   rke2-ingress-nginx-controller-djf74                     1/1     Running     0          24m
kube-system   rke2-ingress-nginx-controller-5qznc                     1/1     Running     0          24m
kube-system   rke2-ingress-nginx-controller-lskud                     1/1     Running     0          24m
kube-system   rke2-ingress-nginx-controller-f7tkk                     1/1     Running     0          131m
kube-system   rke2-ingress-nginx-controller-n9nlj                     1/1     Running     0          120m
kube-system   rke2-ingress-nginx-controller-ww5nj                     1/1     Running     0          12h
kube-system   rke2-metrics-server-c9c78bd66-7t8sr                     1/1     Running     0          12h
kube-system   rke2-snapshot-controller-6f7bbb497d-tdz5j               1/1     Running     0          12h
kube-system   rke2-snapshot-validation-webhook-65b5675d5c-5dpp4       1/1     Running     0          12h

```

# Installing the Rancher Prime server on a Kubernetes cluster

* install helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

* add needed helm charts
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo add jetstack https://charts.jetstack.io
```

* Create a Namespace for Rancher and cert-manager
```
kubectl create namespace cattle-system
kubectl create namespace cert-manager
```

* Apply the certification manager file
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
```
**Link: https://github.com/cert-manager/cert-manager/releases**

* Update repositories
```
helm repo update
```

* Helm install the certificate manager in a new namespace called cert-manager
```
helm install cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.11.0
```

* Verify that the cert-manager pods are running
```
kubectl get pods --namespace cert-manager
```
* Install the latest rancher in the cattle-system namespace and set the hostname
```
helm install rancher rancher-latest/rancher \
--namespace cattle-system \
--set hostname=rancher.example.com
```

* If installed with optional version
```
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.example.com --version 2.7.2
```

* View the status of the deployment
```
kubectl -n cattle-system rollout status deploy/rancher
```
* check password rancher
```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

# Login Rancher

url: https://rancher.example.com/

![login-rancher.png](login-rancher.png?raw=true "login-rancher.png")
