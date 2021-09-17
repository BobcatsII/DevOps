##  Ubuntu20.04 安装Kubeadm

##  参考： https://github.com/cncamp/101/blob/master/1.1.kubeadmin-setup.md

###  基础

```bash
##命令自动补全
打开 /etc/bash.bashrc，找到下列代码，取消注释。
#enable bash completion in interactive shells
#if ! shopt -oq posix; then
# if [-f  /usr/share/bash-completion/bash_completion ]; then
# . /usr/share/bash-completion/bash_completion
# elif [ -f /etc/bash_completion]; then
# . /etc/bash_completion
# fi
#fi

root@paul:/server/kubernetes# source /etc/bash.bashrc
#再执行以下三条命令
# source /usr/share/bash-completion/bash_completion
# source <(kubectl completion bash)
# echo "source <(kubectl completion bash)" >> ~/.bashrc
```



###  第一：配置网卡

```bash
root@paul:~# cat /etc/netplan/00-installer-config.yaml 
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
      - 192.168.56.201/24
      gateway4: 192.168.56.2
      nameservers:
        addresses:
        - 192.168.56.201
  version: 2

##让配置生效
netplan apply

### 配置DNS
sudo apt install resolvconf 
sudo systemctl enable --now resolvconf.service
vim  /etc/resolvconf/resolv.conf.d/head
sudo resolvconf -u
cat /etc/resolv.conf 

##安装sestatus 工具，关闭selinux
##参考https://linuxconfig.org/how-to-disable-enable-selinux-on-ubuntu-20-04-focal-fossa-linux
apt install policycoreutils selinux-utils selinux-basics

root@k8s-master:/server# sestatus 
SELinux status:                 disabled

root@k8s-master:/server# cat /etc/selinux/config 
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
# disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
# default - equivalent to the old strict and targeted policies
# mls     - Multi-Level Security (for military and educational use)
# src     - Custom policy built from source
SELINUXTYPE=default

# SETLOCALDEFS= Check local definition changes
SETLOCALDEFS=0

```



###  第二：配置阿里云ubuntu基础deb源

```bash
root@paul:~# cat /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse


```



###  第三： 配置docker源，并安装运行时

```bash
##添加docker-ce稳定版
root@paul:~# cat /etc/apt/sources.list.d/docker.list
deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu   focal stable


#Update the apt package index and install packages to allow apt to use a repository over HTTPS:
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
#Add Docker’s official GPG key:
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
#Add Docker repositry
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
#Update the apt package index, and install the latest version of Docker Engine and containerd, or go to the next step to install a specific version:
$ sudo apt-get update
$ sudo apt-get install -y docker-ce docker-ce-cli containerd.io


# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]

root@paul:~# docker --version
Docker version 20.10.8, build 3967b7d
```





###  第四： 配置kubernetes 源,并安装kubelet kubeadm kubectl

```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat << EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF  
apt-get update
apt-get install -y kubelet kubeadm kubectl

root@paul:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.1", GitCommit:"632ed300f2c34f6d6d15ca4cef3d3c7073412212", GitTreeState:"clean", BuildDate:"2021-08-19T15:44:22Z", GoVersion:"go1.16.7", Compiler:"gc", Platform:"linux/amd64"}

```



###  第五步：kubeadm初始化

```bash
kubeadm init \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version v1.22.1 \
 --apiserver-advertise-address=192.168.56.201
 
 #配置文件
 root@paul:/server/kubernetes# cat kubeadm.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.22.1
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "192.168.56.201:6443"
networking:
  dnsDomain: cluster.local
  podSubnet: 10.30.0.0/16
  serviceSubnet: 10.96.0.0/12
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs


 
 ###查看镜像版本
 root@paul:/server/kubernetes# kubeadm config images list --config kubeadm.yaml 
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.1
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.1
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.1
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.1
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.4

##预先下载镜像
root@paul:/server/kubernetes# kubeadm config images pull --config kubeadm.yaml 
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
failed to pull image "registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.4": output: Error response from daemon: manifest for registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.4 not found: manifest unknown: manifest unknown
, error: exit status 1
To see the stack trace of this error execute with --v=5 or higher
##拉取coredns镜像失败，手动拉取官网镜像
root@paul:/server/kubernetes# docker pull coredns/coredns
Using default tag: latest
latest: Pulling from coredns/coredns
c6568d217a00: Pull complete 
bc38a22c706b: Pull complete 
Digest: sha256:6e5a02c21641597998b4be7cb5eb1e7b02c0d8d23cce4dd09f4682d463798890
Status: Downloaded newer image for coredns/coredns:latest
docker.io/coredns/coredns:latest

##重新打标签coredns镜像
root@paul:/server/kubernetes# docker tag coredns/coredns:latest registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.4
root@paul:/server/kubernetes# docker images|grep coredns
coredns/coredns                                                               latest    8d147537fb7d   3 months ago   47.6MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                   v1.8.4    8d147537fb7d   3 months ago   47.6MB

###安装kubernetes主节点
kubeadm init  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers  --kubernetes-version v1.22.1  --apiserver-advertise-address=192.168.56.201 --pod-network-cidr 10.30.0.0/16 --service-cidr 10.96.0.0/12

root@paul:/server/kubernetes# kubeadm init --config kubeadm.yaml --upload-certs
[init] Using Kubernetes version: v1.22.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.201]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.201 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.201 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.

##报以上错误的原因是
这是cgroup驱动问题。默认情况下Kubernetes cgroup驱动程序设置为system，但docker设置为systemd。我们需要更改Docker cgroup驱动，通过创建配置文件/etc/docker/daemon.json并添加以下行：

{"exec-opts": ["native.cgroupdriver=systemd"]}
/etc/docker/daemon.json
如果你不懂使用VIM/VI，点击这里寻找更多Vim教程，你也可以使用以下命令创建配置文件，注意下面的命令将会重写你配置文件：

echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' | sudo tee /etc/docker/daemon.json
注意：命令将会重写/etc/docker/daemon.json
然后，为使配置生效，必须重启docker

systemctl daemon-reload
systemctl restart docker

现在，我们尝试重新初始化一个Kubernetes集群，通过运行以下命令。
sudo kubeadm reset
rm /etc/kubernetes -rf
rm /opt/* -rf
rm /var/lib/etcd -rf

##再重新安装一次
root@paul:/server/kubernetes# kubeadm init --config kubeadm.yaml --upload-certs
[init] Using Kubernetes version: v1.22.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.201]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.201 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.201 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 6.002780 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.22" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
b0e09c206b02c4648ef14307a13e1117ccd8d6ff0f4c762521a956e6223d8930
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 3j50bg.1d59j8rzil6s3xhq
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

  kubeadm join 192.168.56.201:6443 --token mp2q48.3wvkn9njps6ftjk6 \
	--discovery-token-ca-cert-hash sha256:2efe3a4bd7c02dfca3a12e81b6b4afe805a0801dff922812890e19477d4cc1aa \
	--control-plane --certificate-key b40b1bc34049061db753315a62a3124806906b602dadaa622de515ae6c646ce1

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.201:6443 --token mp2q48.3wvkn9njps6ftjk6 \
	--discovery-token-ca-cert-hash sha256:2efe3a4bd7c02dfca3a12e81b6b4afe805a0801dff922812890e19477d4cc1aa


#以上这种安装方式有一个问题是Pod网络不通，通过下面这种指定pod和service网络IP，后面部署网络插件就正常
root@k8s-master:/server#  kubeadm init  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers  --kubernetes-version v1.22.1  --apiserver-advertise-address=192.168.56.201 --pod-network-cidr 10.30.0.0/16 --service-cidr 10.96.0.0/12
[init] Using Kubernetes version: v1.22.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.201]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.201 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.56.201 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 6.002183 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.22" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: uancr3.wdyq8okpxs67unxf
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.201:6443 --token uancr3.wdyq8okpxs67unxf \
	--discovery-token-ca-cert-hash sha256:f1c357a93a21b1994b5bf4bd83b225d48e862f8f9dc9a20e50a7bba9e813938d 

```



###  第六：设置环境变量

```bash
root@paul:/server/kubernetes# mkdir -p $HOME/.kube
root@paul:/server/kubernetes# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
root@paul:/server/kubernetes# sudo chown $(id -u):$(id -g) $HOME/.kube/config
root@paul:/server/kubernetes# cat >> /root/.bashrc <<EOF
> export KUBECONFIG=/etc/kubernetes/admin.conf
> EOF
root@paul:/server/kubernetes# source /root/.bashrc
root@paul:/server/kubernetes# kubectl get nodes
NAME         STATUS     ROLES                  AGE     VERSION
k8s-master   NotReady   control-plane,master   5m24s   v1.22.1

```



###  第七：安装Helm

```bash
##官网 https://helm.sh/docs/intro/install/
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

root@paul:/server/kubernetes# helm version
version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"clean", GoVersion:"go1.16.5"}

##配置Repo仓库

```



###  第八： 安装网络插件 cilium

```bash
#https://docs.cilium.io/en/v1.10/gettingstarted/k8s-install-helm/

##添加cilium的repo仓库
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium --version 1.10.4 \
  --namespace kube-system \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=192.168.56.201 \
  --set k8sServicePort=6443
  
 
 ##helm安装不了，使用手动安装
  
##安装cilium-cli
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}

###安装cilium
root@k8s-master:/server/kubernetes# cilium install
ℹ️  using Cilium version "v1.10.4"


###查看cilium是否安装成功
cilium status --wait

###查看网络是否正常
cilium connectivity test
```



###  第九：安装calico网络插件

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```





###  排错

```bash
##
root@k8s-master:/server# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Healthy     ok                                                                                            
etcd-0               Healthy     {"health":"true","reason":""} 

###解决办法  然后将你的scheduler中都port=0注释掉
root@k8s-master:/server# cat /etc/kubernetes/manifests/kube-scheduler.yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
      #- --port=0
    image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.1
    imagePullPolicy: IfNotPresent
root@k8s-master:/server# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""} 

```





###  安装一个应用测试

```bash
##创建一个helm的chart
helm create hello-helm
##安装
helm install hello-nginx hello-helm/
##查看
helm list

root@k8s-master:/server# kubectl get pod
NAME                                    READY   STATUS    RESTARTS   AGE
hello-nginx-hello-helm-f5d6589f-k59tv   1/1     Running   0          13m

```

