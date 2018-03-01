# kubeadm-highavailiability - kubernetes high availiability deployment based on kubeadm, for Kubernetes version 1.9.x/1.7.x/1.6.x

![k8s logo](images/v1.6-v1.7/Kubernetes.png)

- [中文文档(for v1.9.x版本)](README_CN.md)
- [English document(for v1.9.x version)](README.md)
- [中文文档(for v1.7.x版本)](v1.6-v1.7/README_CN.md)
- [English document(for v1.7.x version)](v1.6-v1.7/README.md)
- [中文文档(for v1.6.x版本)](v1.6-v1.7/README_v1.6.x_CN.md)
- [English document(for v1.6.x version)](v1.6-v1.7/README_v1.6.x.md)

---

- [GitHub project URL](https://github.com/cookeem/kubeadm-ha/)
- [OSChina project URL](https://git.oschina.net/cookeem/kubeadm-ha/)

---

- This operation instruction is for version v1.9.x kubernetes cluster

> Before v1.9.0 kubeadm still not support high availability deployment, so it's not recommend for production usage. But from v1.9.0, kubeadm support high availability deployment officially, this instruction version for at least v1.9.0.

### category

1. [deployment architecture](#deployment-architecture)
    1. [deployment architecture summary](#deployment-architecture-summary)
    1. [detail deployment architecture](#detail-deployment-architecture)
    1. [hosts list](#hosts-list)
    1. [ports list](#ports-list)
1. [prerequisites](#prerequisites)
    1. [version info](#version-info)
    1. [required docker images](#required-docker-images)
    1. [system configuration](#system-configuration)
1. [kubernetes installation](#kubernetes-installation)
    1. [kubernetes and related services installation](#kubernetes-and-related-services-installation)
1. [configuration files settings](#configuration-files-settings)
    1. [script files settings](#script-files-settings) 
    1. [deploy independent etcd cluster](#deploy-independent-etcd-cluster)
1. [use kubeadm to init first master](#use-kubeadm-to-init-first-master)
    1. [kubeadm init](#kubeadm-init)
    1. [basic components installation](#basic-components-installation)
1. [kubernetes masters high avialiability configuration](#kubernetes-masters-high-avialiability-configuration)
    1. [copy configuration files](#copy-configuration-files)
    1. [other master nodes init](#other-master-nodes-init)
    1. [keepalived installation](#keepalived-installation)
    1. [nginx load balancer configuration](#nginx-load-balancer-configuration)
    1. [kube-proxy configuration](#kube-proxy-configuration)
1. [all nodes join the kubernetes cluster](#all-nodes-join-the-kubernetes-cluster)
    1. [use kubeadm to join the cluster](#use-kubeadm-to-join-the-cluster)
    1. [verify kubernetes cluster high availiablity](#verify-kubernetes-cluster-high-availiablity)
    

### deployment architecture

#### deployment architecture summary

![ha logo](images/v1.6-v1.7/ha.png)

---

[category](#category)

#### detail deployment architecture

![k8s ha](images/v1.6-v1.7/k8s-ha.png)

* kubernetes components:

> kube-apiserver: exposes the Kubernetes API. It is the front-end for the Kubernetes control plane. It is designed to scale horizontally – that is, it scales by deploying more instances.

> etcd: is used as Kubernetes’ backing store. All cluster data is stored here. Always have a backup plan for etcd’s data for your Kubernetes cluster.


> kube-scheduler: watches newly created pods that have no node assigned, and selects a node for them to run on.


> kube-controller-manager: runs controllers, which are the background threads that handle routine tasks in the cluster. Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.

> kubelet: is the primary node agent. It watches for pods that have been assigned to its node (either by apiserver or via local configuration file)

> kube-proxy: enables the Kubernetes service abstraction by maintaining network rules on the host and performing connection forwarding.


* load balancer

> keepalived cluster config a virtual IP address (192.168.20.10), this virtual IP address point to k8s-master01, k8s-master02, k8s-master03. 

> nginx service as the load balancer of k8s-master01, k8s-master02, k8s-master03's apiserver. The other nodes kubernetes services connect the keepalived virtual ip address (192.168.20.10) and nginx exposed port (16443) to communicate with the master cluster's apiservers. 

---

[category](#category)

#### hosts list

HostName | IPAddress | Notes | Components 
:--- | :--- | :--- | :---
k8s-master01 ~ 03 | 192.168.20.27 ~ 29 | master nodes * 3 | keepalived, nginx, etcd, kubelet, kube-apiserver, kube-scheduler, kube-proxy, kube-dashboard, heapster, calico
N/A | 192.168.20.10 | keepalived virtual IP | N/A
node01 ~ 04 | 192.168.20.17 ~ 20 | worker nodes * 4 | kubelet, kube-proxy

#### ports list

- master ports list

Protocol | Direction | Port | Comment
:--- | :--- | :--- | :---
TCP | Inbound | 16443*    | Load balancer Kubernetes API server port
TCP | Inbound | 6443*     | Kubernetes API server
TCP | Inbound | 2379-2380 | etcd server client API
TCP | Inbound | 10250     | Kubelet API
TCP | Inbound | 10251     | kube-scheduler
TCP | Inbound | 10252     | kube-controller-manager
TCP | Inbound | 10255     | Read-only Kubelet API

- worker ports list

Protocol | Direction | Port | Comment
:--- | :--- | :--- | :---
TCP | Inbound | 10250       | Kubelet API
TCP | Inbound | 10255       | Read-only Kubelet API
TCP | Inbound | 30000-32767 | NodePort Services**

---

[category](#category)

### prerequisites

#### version info

* Linux version: CentOS 7.4.1708

```
$ cat /etc/redhat-release 
CentOS Linux release 7.4.1708 (Core) 
```

* docker version: 18.02.0-ce

```
$ docker version
Client:
 Version:	18.02.0-ce
 API version:	1.36
 Go version:	go1.9.3
 Git commit:	fc4de44
 Built:	Wed Feb  7 21:14:12 2018
 OS/Arch:	linux/amd64
 Experimental:	false
 Orchestrator:	swarm

Server:
 Engine:
  Version:	18.02.0-ce
  API version:	1.36 (minimum version 1.12)
  Go version:	go1.9.3
  Git commit:	fc4de44
  Built:	Wed Feb  7 21:17:42 2018
  OS/Arch:	linux/amd64
  Experimental:	false
```

* kubeadm version: v1.9.3

```
$ kubeadm --version
kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T11:55:20Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
```

* kubelet version: v1.9.3

```
$ kubelet --version
Kubernetes v1.9.3
```

* networks add-ons

> flannel

> calico

---

[category](#category)

#### required docker images

```
$ docker pull quay.io/calico/kube-controllers:v2.0.0
$ docker pull quay.io/calico/node:v3.0.1
$ docker pull quay.io/calico/cni:v2.0.0
$ docker pull quay.io/coreos/flannel:v0.9.1-amd64
$ docker pull gcr.io/google_containers/heapster-amd64:v1.4.2
$ docker pull gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
$ docker pull gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
$ docker pull gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
$ docker pull gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
$ docker pull gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
$ docker pull gcr.io/google_containers/kube-apiserver-amd64:v1.9.3
$ docker pull gcr.io/google_containers/kube-controller-manager-amd64:v1.9.3
$ docker pull gcr.io/google_containers/kube-proxy-amd64:v1.9.3
$ docker pull gcr.io/google_containers/kube-scheduler-amd64:v1.9.3
$ docker pull gcr.io/google_containers/kubernetes-dashboard-amd64:v1.8.1
$ docker pull nginx
```

---

[category](#category)

#### system configuration

* on all kubernetes nodes: add kubernetes' repository

```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

* on all kubernetes nodes: use yum to update system

```
$ yum update -y
```

* on all kubernetes nodes: turn off firewalld service

```
$ systemctl disable firewalld && systemctl stop firewalld && systemctl status firewalld
```

* on all kubernetes nodes: set SELINUX to permissive mode

```
$ vi /etc/selinux/config
SELINUX=permissive

$ setenforce 0
```

* on all kubernetes nodes: set iptables parameters

```
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

* on all kubernetes nodes: disable swap

```
$ swapoff -a

# disable swap mount point in /etc/fstab
$ vi /etc/fstab
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

# check swap is disabled
$ cat /proc/swaps
Filename                Type        Size    Used    Priority
```

* on all kubernetes nodes: reboot host

```
$ reboot
```

---

[category](#category)

### kubernetes installation

#### kubernetes and related services installation

* on all kubernetes nodes: check SELINUX mode, it must be permissive mode

```
$ getenforce
Permissive
```

* on all kubernetes nodes: install kubernetes and related services, then start up kubelet and docker daemon

```
$ curl -fsSL https://get.docker.com/ | sh
$ yum install -y docker-compose-1.9.0-5.el7.noarch
$ systemctl enable docker && systemctl start docker

# solve problem https://ask.openstack.org/en/question/110437/importerror-cannot-import-name-unrewindablebodyerror/
$ pip uninstall requests
$ pip uninstall urllib3
$ yum remove python-urllib3 -y
$ yum remove python-requests -y
$ yum install python-urllib3
$ yum install python-requests

$ yum install -y kubelet-1.9.3-0.x86_64 kubeadm-1.9.3-0.x86_64 kubectl-1.9.3-0.x86_64
$ systemctl enable kubelet && systemctl start kubelet
```

* on all kubernetes nodes: set kubelet KUBELET_CGROUP_ARGS parameter the same as docker daemon's settings, here docker daemon and kubelet use cgroupfs as cgroup-driver.

```
# by default kubelet use cgroup-driver=systemd, modify it as cgroup-driver=cgroupfs
$ vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
#Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"

# reload then restart kubelet service
$ systemctl daemon-reload && systemctl restart kubelet
```

* on all kubernetes nodes: install and start keepalived service

```
$ yum install -y keepalived
$ systemctl enable keepalived && systemctl restart keepalived
```

---

[category](#category)

### configuration files settings

#### script files settings

* on all kubernetes master nodes: get the source code, and change the working directory to the source code directory

```
$ git clone https://github.com/cookeem/kubeadm-ha

$ cd kubeadm-ha
```

* on all kubernetes master nodes: set the `create-config.sh` file, this script will create all configuration files, follow the setting comment and make sure you set the parameters correctly.

```
$ vi create-config.sh

# local machine ip address
export K8SHA_IPLOCAL=192.168.20.27

# local machine etcd name, options: etcd1, etcd2, etcd3
export K8SHA_ETCDNAME=etcd1

# local machine keepalived state config, options: MASTER, BACKUP. One keepalived cluster only one MASTER, other's are BACKUP
export K8SHA_KA_STATE=MASTER

# local machine keepalived priority config, options: 102, 101, 100. MASTER must 102
export K8SHA_KA_PRIO=102

# local machine keepalived network interface name config, for example: eth0
export K8SHA_KA_INTF=nm-bond

#######################################
# all masters settings below must be same
#######################################

# master keepalived virtual ip address
export K8SHA_IPVIRTUAL=192.168.20.10

# k8s-master01 ip address
export K8SHA_IP1=192.168.20.27

# k8s-master02 ip address
export K8SHA_IP2=192.168.20.28

# k8s-master03 ip address
export K8SHA_IP3=192.168.20.29

# k8s-master01 hostname
export K8SHA_HOSTNAME1=k8s-master01

# k8s-master02 hostname
export K8SHA_HOSTNAME2=k8s-master02

# k8s-master03 hostname
export K8SHA_HOSTNAME3=k8s-master03

# keepalived auth_pass config, all masters must be same
export K8SHA_KA_AUTH=4cdf7dc3b4c90194d1600c483e10ad1d

# kubernetes cluster token, you can use 'kubeadm token generate' to get a new one
export K8SHA_TOKEN=7f276c.0741d82a5337f526

# kubernetes CIDR pod subnet, if CIDR pod subnet is "10.244.0.0/16" please set to "10.244.0.0\\/16"
export K8SHA_CIDR=10.244.0.0\\/16

# calico network settings, set a reachable ip address for the cluster network interface, for example you can use the gateway ip address
export K8SHA_CALICO_REACHABLE_IP=192.168.20.1
```

* on all kubernetes master nodes: run the `create-config.sh` script file and create related configuration files:

> etcd cluster docker-compose.yaml file

> keepalived configuration file

> nginx load balancer docker-compose.yaml file

> kubeadm init configuration file

> calico configuration file

```
$ ./create-config.sh
set etcd cluster docker-compose.yaml file success: etcd/docker-compose.yaml
set keepalived config file success: /etc/keepalived/keepalived.conf
set nginx load balancer config file success: nginx-lb/nginx-lb.conf
set kubeadm init config file success: kubeadm-init.yaml
set calico deployment config file success: kube-calico/calico.yaml
```

---

[category](#category)

#### deploy independent etcd cluster

* on all kubernetes master nodes: deploy independent etcd cluster (non-TLS mode)

```
# reset kubernetes cluster
$ kubeadm reset

# clear etcd cluster data
$ rm -rf /var/lib/etcd-cluster

# reset and start etcd cluster
$ docker-compose --file etcd/docker-compose.yaml stop
$ docker-compose --file etcd/docker-compose.yaml rm -f
$ docker-compose --file etcd/docker-compose.yaml up -d

# check etcd cluster status
$ docker exec -ti etcd etcdctl cluster-health
member 531504c79088f553 is healthy: got healthy result from http://192.168.20.29:2379
member 56c53113d5e1cfa3 is healthy: got healthy result from http://192.168.20.27:2379
member 7026e604579e4d64 is healthy: got healthy result from http://192.168.20.28:2379
cluster is healthy

$ docker exec -ti etcd etcdctl member list
531504c79088f553: name=etcd3 peerURLs=http://192.168.20.29:2380 clientURLs=http://192.168.20.29:2379,http://192.168.20.29:4001 isLeader=false
56c53113d5e1cfa3: name=etcd1 peerURLs=http://192.168.20.27:2380 clientURLs=http://192.168.20.27:2379,http://192.168.20.27:4001 isLeader=false
7026e604579e4d64: name=etcd2 peerURLs=http://192.168.20.28:2380 clientURLs=http://192.168.20.28:2379,http://192.168.20.28:4001 isLeader=true
```

---

[category](#category)

### use kubeadm to init first master

#### kubeadm init

* on all kubernetes master nodes: reset cni and docker network

```
$ systemctl stop kubelet
$ systemctl stop docker
$ rm -rf /var/lib/cni/
$ rm -rf /var/lib/kubelet/*
$ rm -rf /etc/cni/

$ ip a | grep -E 'docker|flannel|cni'
$ ip link del docker0
$ ip link del flannel.1
$ ip link del cni0

$ systemctl restart docker && systemctl restart kubelet
$ ip a | grep -E 'docker|flannel|cni'
```

* on k8s-master01: use kubeadm to init a kubernetes cluster, notice: you must save the following message: kubeadm join --token XXX --discovery-token-ca-cert-hash YYY , this command will use lately.

```
$ kubeadm init --config=kubeadm-init.yaml
...
  kubeadm join --token 7f276c.0741d82a5337f526 192.168.20.27:6443 --discovery-token-ca-cert-hash sha256:a4a1eaf725a0fc67c3028b3063b92e6af7f2eb0f4ae028f12b3415a6fd2d2a5e
```

* on all kubernetes master nodes: set kubectl client environment variable

```
$ vi ~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf

$ source ~/.bashrc
```

#### basic components installation

* on k8s-master01: install flannel network add-ons

```
# master may not work if no network add-ons
$ kubectl get node
NAME              STATUS    ROLES     AGE       VERSION
k8s-master01   NotReady  master    14s       v1.9.3

# install flannel add-ons
$ kubectl apply -f kube-flannel/
clusterrole "flannel" created
clusterrolebinding "flannel" created
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created

# waiting for all pods to be normal status
$ kubectl get pods --all-namespaces -o wide -w
```

* on k8s-master01: install calico network add-ons

```
# set master node as schedulable
$ kubectl taint nodes --all node-role.kubernetes.io/master-

# install calico network add-ons
$ kubectl apply -f kube-calico/
configmap "calico-config" created
secret "calico-etcd-secrets" created
daemonset "calico-node" created
deployment "calico-kube-controllers" created
serviceaccount "calico-kube-controllers" created
serviceaccount "calico-node" created
clusterrole "calico-kube-controllers" created
clusterrolebinding "calico-kube-controllers" created
clusterrole "calico-node" created
clusterrolebinding "calico-node" created
```

* on k8s-master01: install dashboard

```
$ kubectl apply -f kube-dashboard/
serviceaccount "admin-user" created
clusterrolebinding "admin-user" created
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created

$ kubectl get pods --all-namespaces
NAMESPACE       NAME                                        READY     STATUS    RESTARTS   AGE
kube-system     calico-kube-controllers-7749c84f4-p8c4d     1/1       Running   0          3m
kube-system     calico-node-2jlwj                           2/2       Running   6          13m
kube-system     kube-apiserver-k8s-master01              1/1       Running   6          5m
kube-system     kube-controller-manager-k8s-master01     1/1       Running   8          5m
kube-system     kube-dns-6f4fd4bdf-8jnpc                    3/3       Running   3          4m
kube-system     kube-flannel-ds-2fgsw                       1/1       Running   8          14m
kube-system     kube-proxy-7rh8x                            1/1       Running   3          13m
kube-system     kube-scheduler-k8s-master01              1/1       Running   8          5m
kube-system     kubernetes-dashboard-87497878f-p6nj4        1/1       Running   0          4m
```

* use browser to access dashboard

> https://k8s-master01:30000/#!/login

* dashboard login interface

![dashboard-login](images/dashboard-login.png)

* use command below to get token, copy and paste the token on login interface 

```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

![dashboard](images/dashboard.png)

* on k8s-master01: install heapster

```
$ kubectl apply -f kube-heapster/influxdb/
service "monitoring-grafana" created
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created

$ kubectl apply -f kube-heapster/rbac/
clusterrolebinding "heapster" created

$ kubectl get pods --all-namespaces 
NAME                                      READY     STATUS        RESTARTS   AGE
calico-kube-controllers-7749c84f4-p8c4d   1/1       Running       0          8m
calico-node-2jlwj                         2/2       Running       6          13d
heapster-698c5f45bd-wnv6x                 1/1       Running       0          1m
kube-apiserver-k8s-master01            1/1       Running       6          5d
kube-controller-manager-k8s-master01   1/1       Running       8          5d
kube-dns-6f4fd4bdf-8jnpc                  3/3       Running       3          4h
kube-flannel-ds-2fgsw                     1/1       Running       8          14d
kube-proxy-7rh8x                          1/1       Running       3          13d
kube-scheduler-k8s-master01            1/1       Running       8          5d
kubernetes-dashboard-87497878f-p6nj4      1/1       Running       0          4h
monitoring-grafana-5ffb49ff84-xxwzn       1/1       Running       0          1m
monitoring-influxdb-5b77d47fdd-wd7xm      1/1       Running       0          1m

# wait for 5 minutes
kubectl top pod --all-namespaces
NAMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)   
kube-system   calico-kube-controllers-d987c6db5-zjxnv   0m           20Mi            
kube-system   calico-node-hmdlg                         16m          83Mi            
kube-system   heapster-dfd674df9-hct67                  1m           24Mi            
kube-system   kube-apiserver-k8s-master01                   24m          240Mi           
kube-system   kube-controller-manager-k8s-master01          14m          50Mi            
kube-system   kube-dns-6f4fd4bdf-zg66x                  1m           49Mi            
kube-system   kube-flannel-ds-h7ng4                     6m           33Mi            
kube-system   kube-proxy-mxcwz                          2m           29Mi            
kube-system   kube-scheduler-k8s-master01                   5m           22Mi            
kube-system   kubernetes-dashboard-7b7b5cd79b-6ldfn     0m           20Mi            
kube-system   monitoring-grafana-76848b566c-h5998       0m           28Mi            
kube-system   monitoring-influxdb-6c4b84d695-whzmp      1m           24Mi            
```

* heapster performance info will show on dashboard

> https://k8s-master01:30000/#!/login

![heapster-dashboard](images/heapster-dashboard.png)

![heapster](images/heapster.png)

* now flannel, calico, dashboard, heapster had installed on the first master node

---

[category](#category)

### kubernetes masters high avialiability configuration

#### copy configuration files

* on k8s-master01: copy `category/etc/kubernetes/pki` to k8s-master02 and k8s-master03

```
scp -r /etc/kubernetes/pki k8s-master02:/etc/kubernetes/

scp -r /etc/kubernetes/pki k8s-master03:/etc/kubernetes/
```

---
[category](#category)

#### other master nodes init

* on k8s-master02: use kubeadm to init master cluster

```
# you will found that output token and discovery-token-ca-cert-hash are the same with k8s-master01
$ kubeadm init --config=kubeadm-init.yaml
...
  kubeadm join --token 7f276c.0741d82a5337f526 192.168.20.28:6443 --discovery-token-ca-cert-hash sha256:a4a1eaf725a0fc67c3028b3063b92e6af7f2eb0f4ae028f12b3415a6fd2d2a5e
```

* on k8s-master03: use kubeadm to init master cluster

```
# you will found that output token and discovery-token-ca-cert-hash are the same with k8s-master01
$ kubeadm init --config=kubeadm-init.yaml
...
  kubeadm join --token 7f276c.0741d82a5337f526 192.168.20.29:6443 --discovery-token-ca-cert-hash sha256:a4a1eaf725a0fc67c3028b3063b92e6af7f2eb0f4ae028f12b3415a6fd2d2a5e
```

* on any kubernetes master nodes: check nodes status

```
$ kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
k8s-master01          Ready     master    19m       v1.9.3
k8s-master02          Ready     master    4m        v1.9.3
k8s-master03          Ready     master    4m        v1.9.3
```

* on all kubernetes master nodes: add apiserver-count settings in `/etc/kubernetes/manifests/kube-apiserver.yaml` file

```
$ vi /etc/kubernetes/manifests/kube-apiserver.yaml
    - --apiserver-count=3

# restart service
$ systemctl restart docker && systemctl restart kubelet
```

* on any kubernetes master nodes: check all pod status

```
$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                      READY     STATUS    RESTARTS   AGE       IP              NODE
kube-system   calico-kube-controllers-d987c6db5-zjxnv   1/1       Running   2          14m       192.168.20.27   k8s-master01
kube-system   calico-node-dldxz                         2/2       Running   2          3m        192.168.20.29   k8s-master03
kube-system   calico-node-hmdlg                         2/2       Running   4          14m       192.168.20.27   k8s-master01
kube-system   calico-node-tkbbx                         2/2       Running   2          3m        192.168.20.28   k8s-master02
kube-system   heapster-dfd674df9-hct67                  1/1       Running   2          11m       10.244.172.11   k8s-master01
kube-system   kube-apiserver-k8s-master01                   1/1       Running   1          2m        192.168.20.27   k8s-master01
kube-system   kube-apiserver-k8s-master02                   1/1       Running   1          2m        192.168.20.28   k8s-master02
kube-system   kube-apiserver-k8s-master03            1/1       Running   0          24s       192.168.20.29   k8s-master03
kube-system   kube-controller-manager-k8s-master01          1/1       Running   2          15m       192.168.20.27   k8s-master01
kube-system   kube-controller-manager-k8s-master02          1/1       Running   1          2m        192.168.20.28   k8s-master02
kube-system   kube-controller-manager-k8s-master03          1/1       Running   1          2m        192.168.20.29   k8s-master03
kube-system   kube-dns-6f4fd4bdf-zg66x                  3/3       Running   6          16m       10.244.172.13   k8s-master01
kube-system   kube-flannel-ds-6njgf                     1/1       Running   1          3m        192.168.20.29   k8s-master03
kube-system   kube-flannel-ds-g24ww                     1/1       Running   1          3m        192.168.20.28   k8s-master02
kube-system   kube-flannel-ds-h7ng4                     1/1       Running   2          16m       192.168.20.27   k8s-master01
kube-system   kube-proxy-2kk8s                          1/1       Running   1          3m        192.168.20.28   k8s-master02
kube-system   kube-proxy-mxcwz                          1/1       Running   2          16m       192.168.20.27   k8s-master01
kube-system   kube-proxy-vz7nf                          1/1       Running   1          3m        192.168.20.29   k8s-master03
kube-system   kube-scheduler-k8s-master01                   1/1       Running   2          16m       192.168.20.27   k8s-master01
kube-system   kube-scheduler-k8s-master02                   1/1       Running   1          2m        192.168.20.28   k8s-master02
kube-system   kube-scheduler-k8s-master03                   1/1       Running   1          2m        192.168.20.29   k8s-master03
kube-system   kubernetes-dashboard-7b7b5cd79b-6ldfn     1/1       Running   3          12m       10.244.172.12   k8s-master01
kube-system   monitoring-grafana-76848b566c-h5998       1/1       Running   2          11m       10.244.172.14   k8s-master01
kube-system   monitoring-influxdb-6c4b84d695-whzmp      1/1       Running   2          11m       10.244.172.10   k8s-master01
```

* on any kubernetes master nodes: set all master nodes scheduable

```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
node "k8s-master02" untainted
node "k8s-master03" untainted
```

* on any kubernetes master nodes: scale the kube-system deployment to all master nodes

```
$ kubectl get deploy -n kube-system
NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
calico-kube-controllers   1         1         1            1           14d
heapster                  1         1         1            0           8m
kube-dns                  3         3         3            3           14d
kubernetes-dashboard      1         1         1            1           14d
monitoring-grafana        1         1         1            0           8m
monitoring-influxdb       1         1         1            0           8m

# calico scale to all master nodes
$ kubectl scale --replicas=3 -n kube-system deployment/calico-kube-controllers
$ kubectl get pods --all-namespaces -o wide| grep calico-kube-controllers

# dns scale to all master nodes
$ kubectl scale --replicas=3 -n kube-system deployment/kube-dns
$ kubectl get pods --all-namespaces -o wide| grep kube-dns

# dashboard scale to all master nodes
$ kubectl scale --replicas=3 -n kube-system deployment/kubernetes-dashboard
$ kubectl get pods --all-namespaces -o wide| grep kubernetes-dashboard
```

```
kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
k8s-master01          Ready     master    38m       v1.9.3
k8s-master02          Ready     master    25m       v1.9.3
k8s-master03          Ready     master    25m       v1.9.3
```

---

[category](#category)

#### keepalived installation

* on all kubernetes master nodes: install keepalived service

```
$ systemctl restart keepalived

$ ping 192.168.20.10
```

---

[category](#category)

#### nginx load balancer configuration

* on all kubernetes master nodes: install nginx load balancer

```
$ docker-compose -f nginx-lb/docker-compose.yaml up -d
```

* on all kubernetes master nodes: check nginx load balancer and keepalived

```
curl -k 192.168.20.10:16443 | wc -l
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    14    0    14    0     0   3958      0 --:--:-- --:--:-- --:--:-- 14000
1
```

---

[category](#category)

#### kube-proxy configuration

- on any kubernetes master nodes: set kube-proxy server settings, make sure this settings use the keepalived virtual IP and nginx load balancer port (here is: https://192.168.20.10:16443)

```
$ kubectl edit -n kube-system configmap/kube-proxy
        server: https://192.168.20.10:16443
```

- on any kubernetes master nodes: delete all kube-proxy pod to restart it

```
$ kubectl get pods --all-namespaces -o wide | grep proxy

$ kubectl delete pod -n kube-system kube-proxy-XXX
```

---

[category](#category)

### all nodes join the kubernetes cluster

#### use kubeadm to join the cluster

- on all kubernetes worker nodes: use kubeadm to join the cluster, here we use the k8s-master01 apiserver address and port.

```
$ kubeadm join --token 7f276c.0741d82a5337f526 192.168.20.27:6443 --discovery-token-ca-cert-hash sha256:a4a1eaf725a0fc67c3028b3063b92e6af7f2eb0f4ae028f12b3415a6fd2d2a5e
```

- on all kubernetes worker nodes: set the `/etc/kubernetes/bootstrap-kubelet.conf` server settings, make sure this settings use the keepalived virtual IP and nginx load balancer port (here is: https://192.168.20.10:16443)

```
sed -e "s/192.168.20.27:6443/192.168.20.10:16443/g" /etc/kubernetes/bootstrap-kubelet.conf > /etc/kubernetes/bootstrap-kubelet.conf

systemctl restart docker && systemctl restart kubelet
```


```
kubectl get nodes
NAME              STATUS    ROLES     AGE       VERSION
k8s-master01          Ready     master    46m       v1.9.3
k8s-master02          Ready     master    44m       v1.9.3
k8s-master03          Ready     master    44m       v1.9.3
node01            Ready     <none>    50s       v1.9.3
node02            Ready     <none>    26s       v1.9.3
node03            Ready     <none>    22s       v1.9.3
node04            Ready     <none>    17s       v1.9.3
```

- on any kubernetes master nodes: set the worker nodes labels

```
kubectl label nodes node01 role=worker
kubectl label nodes node02 role=worker
kubectl label nodes node03 role=worker
kubectl label nodes node04 role=worker
```

#### verify kubernetes cluster high availiablity

```
# create a nginx deployment, replicas=3
$ kubectl run nginx --image=nginx --replicas=3 --port=80
deployment "nginx" created

# check nginx pod status
$ kubectl get pods -l=run=nginx -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP              NODE
nginx-6c7c8978f5-558kd   1/1       Running   0          9m        10.244.77.217   node03
nginx-6c7c8978f5-ft2z5   1/1       Running   0          9m        10.244.172.67   k8s-master01
nginx-6c7c8978f5-jr29b   1/1       Running   0          9m        10.244.85.165   node04

# create nginx NodePort service
$ kubectl expose deployment nginx --type=NodePort --port=80
service "nginx" exposed

# check nginx service status
$ kubectl get svc -l=run=nginx -o wide
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE       SELECTOR
nginx     NodePort   10.101.144.192   <none>        80:30847/TCP   10m       run=nginx

# check nginx NodePort service accessibility
$ curl k8s-master01:30847
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

- now kubernetes high availiability cluster setup successfully 😃
