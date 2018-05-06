---
title: kubernetes-installation
tags: kubernetes
date: 2018-05-06 14:32:31
---

__基于Kubernetes1.10.2__

### 0x00 前置条件

+ ubuntu16.04 虚拟机
+ shadowsocks 科学上网

本机是Win10，通过Vmvare安装好几台ubuntu16.04虚拟机，均使用root账号登陆操作，本机已经可以通过shadowsocks(简称ss)进行科学上网

#### 虚拟机列表

|主机名|ip|角色|配置|OS|
|---|---|---|---|---|
|K8s-1|192.168.5.136|master|2U2G|Ubuntu16.04|
|K8s-2|192.168.5.137|node|1U2G|Ubuntu16.04|
|K8s-3|192.168.5.138|node|1U2G|Ubuntu16.04|

### 0x01 准备环境(所有虚拟机)

#### ubuntu虚拟机配置apt源

国内配置apt源为阿里云的源下载速度会快一些，(shadowsocks设置为PAC模式)

```console
$ cp -p /etc/apt/sources.list /etc/apt/sources.list.bak

$ cat <<-EOF | sudo tee /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
EOF

$ apt-get update
```

#### ubuntu虚拟机配置代理进行科学上网

+ 首先要在本机的shadowsocks上使能局域网访问(右键shadowsocks进行选择)

+ 在ubuntu虚拟机上配置全局代理，`proxyserveraddr`为本机的IP地址，Windows上可使用`ipconfig`查看，`proxyserverport`为shadowsocks端口，默认为1080

    将下面的内容添加到`~/.profile`末尾
    ```bash
    export proxyserveraddr=192.168.99.248
    export proxyserverport=1080
    export HTTP_PROXY="http://$proxyserveraddr:$proxyserverport/"
    export HTTPS_PROXY="https://$proxyserveraddr:$proxyserverport/"
    export FTP_PROXY="ftp://$proxyserveraddr:$proxyserverport/"
    export SOCKS_PROXY="socks://$proxyserveraddr:$proxyserverport/"
    export NO_PROXY="localhost,127.0.0.1,localaddress,.localdomain.com,10.0.0.0/8,192.168.0.0/16"
    export http_proxy="http://$proxyserveraddr:$proxyserverport/"
    export https_proxy="https://$proxyserveraddr:$proxyserverport/"
    export ftp_proxy="ftp://$proxyserveraddr:$proxyserverport/"
    export socks_proxy="socks://$proxyserveraddr:$proxyserverport/"
    export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com,10.0.0.0/8,192.168.0.0/16"
    ```

    执行下面的命令生效proxy配置
    ```console
    $ source ~/.profile

    $ cat <<-EOF| sudo tee /etc/apt/apt.conf
    Acquire::http::proxy "http://$proxyserveraddr:$proxyserverport/";
    Acquire::https::proxy "https://$proxyserveraddr:$proxyserverport/";
    Acquire::ftp::proxy "ftp://$proxyserveraddr:$proxyserverport/";
    Acquire::socks::proxy "socks://$proxyserveraddr:$proxyserverport/";
    EOF
    ```

+ 测试proxy是否可用

    ```console
    $ curl www.google.com
    ```

#### 关闭swap

编辑`/etc/fstab`文件，注释掉引用swap的行，保存并重启后输入`sudo swapoff -a`即可。参考[Kubelet/Kubernetes should work with Swap Enabled](https://github.com/kubernetes/kubernetes/issues/53533)

<!-- more -->

### 0x02 安装Docker(所有虚拟机)

#### 配置好docker源

*版本： 17.03.0~ce-0~ubuntu-xenial*

```console
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

#### 安装Docker CE

```console
$ sudo apt-get update

$ sudo apt-get install docker-ce=17.03.0~ce-0~ubuntu-xenial
```

详细信息参考[官方文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce)

### 0x03 安装Kubernetes(所有虚拟机)

#### 配置docker镜像加速

参考[阿里云镜像加速器](https://cr.console.aliyun.com/#/accelerator)

#### 安装kubeadm, kubelet 和 kubectl

```console
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

$ apt-get update

$ apt-get install -y kubelet kubeadm kubectl
```

#### 提前pull镜像

由于网络原因，我们需要提前拉取k8s初始化需要用到的镜像，并添加对应的`k8s.gcr.io`标签

```bash
## 拉取镜像
docker pull reg.qiniu.com/k8s/kube-apiserver-amd64:v1.10.2
docker pull reg.qiniu.com/k8s/kube-controller-manager-amd64:v1.10.2
docker pull reg.qiniu.com/k8s/kube-scheduler-amd64:v1.10.2
docker pull reg.qiniu.com/k8s/kube-proxy-amd64:v1.10.2
docker pull reg.qiniu.com/k8s/etcd-amd64:3.1.12
docker pull reg.qiniu.com/k8s/pause-amd64:3.1

## 添加Tag
docker tag reg.qiniu.com/k8s/kube-apiserver-amd64:v1.10.2 k8s.gcr.io/kube-apiserver-amd64:v1.10.2
docker tag reg.qiniu.com/k8s/kube-scheduler-amd64:v1.10.2 k8s.gcr.io/kube-scheduler-amd64:v1.10.2
docker tag reg.qiniu.com/k8s/kube-controller-manager-amd64:v1.10.2 k8s.gcr.io/kube-controller-manager-amd64:v1.10.2
docker tag reg.qiniu.com/k8s/kube-proxy-amd64:v1.10.2 k8s.gcr.io/kube-proxy-amd64:v1.10.2
docker tag reg.qiniu.com/k8s/etcd-amd64:3.1.12 k8s.gcr.io/etcd-amd64:3.1.12
docker tag reg.qiniu.com/k8s/pause-amd64:3.1 k8s.gcr.io/pause-amd64:3.1

## 在Kubernetes 1.10 中，增加了CoreDNS，如果使用CoreDNS(默认关闭)，则不需要下面三个镜像。
docker pull reg.qiniu.com/k8s/k8s-dns-sidecar-amd64:1.14.10
docker pull reg.qiniu.com/k8s/k8s-dns-kube-dns-amd64:1.14.10
docker pull reg.qiniu.com/k8s/k8s-dns-dnsmasq-nanny-amd64:1.14.10

docker tag reg.qiniu.com/k8s/k8s-dns-sidecar-amd64:1.14.10 k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.10
docker tag reg.qiniu.com/k8s/k8s-dns-kube-dns-amd64:1.14.10 k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.10
docker tag reg.qiniu.com/k8s/k8s-dns-dnsmasq-nanny-amd64:1.14.10 k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.10
```

需要的镜像是`/etc/kubernetes/manifests/`目录下查看各个yaml文件中汇总得到的

### 0x04 配置master节点

#### 使用kubeadm init初始化master节点

```console
$ kubeadm init --apiserver-advertise-address=192.168.5.136 --kubernetes-version=v1.10.2 --feature-gates=CoreDNS=true --pod-network-cidr=192.168.0.0/16
```

init 常用主要参数：

+ --kubernetes-version: 指定Kubenetes版本，如果不指定该参数，会从google网站下载最新的版本信息。
+ --pod-network-cidr: 指定pod网络的IP地址范围，它的值取决于你在下一步选择的哪个网络网络插件，比如我在本文中使用的是Calico网络，需要指定为192.168.0.0/16。
+ --apiserver-advertise-address: 指定master服务发布的Ip地址，如果不指定，则会自动检测网络接口，通常是内网IP。
+ --feature-gates=CoreDNS: 是否使用CoreDNS，值为true/false，CoreDNS插件在1.10中提升到了Beta阶段，最终会成为Kubernetes的缺省选项

最终输出：

```console
root@K8s-1:~# kubeadm init --apiserver-advertise-address=192.168.5.136 --kubernetes-version=v1.10.2 --feature-gates=CoreDNS=true --pod-network-cidr=192.168.0.0/16
[init] Using Kubernetes version: v1.10.2
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[preflight] Starting the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [k8s-1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.5.136]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [k8s-1] and IPs [192.168.5.136]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 24.001741 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node k8s-1 as master by adding a label and a taint
[markmaster] Master k8s-1 tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: dfzpd9.7xxx9kre811fgw1n
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.5.136:6443 --token dfzpd9.7xxx9kre811fgw1n --discovery-token-ca-cert-hash sha256:31d53cf706073f557b43e5e7acacb47dceb828ccced3c648dbfbfa4d86d74b1c

```

kubeadm init 输出的token用于master和加入节点间的身份认证，token是机密的，需要保证它的安全，因为拥有此标记的人都可以随意向集群中添加节点。

如果想在非root用户下使用kubectl，可以执行如下命令

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 验证部署情况

在浏览器中输入`https://<master-ip>:6443`来验证一下是否部署成功，我这里master-ip是192.168.5.136， 返回如下：

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```

#### 安装网络插件

安装一个网络插件是必须的，因为你的pods之间需要彼此通信。

网络部署必须是优先于任何应用的部署，详细的网络列表可参考[插件页面](https://kubernetes.io/docs/concepts/cluster-administration/addons/)。

本文使用的是[Calico网络](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/)，安装如下：

```bash
# 使用国内镜像
kubectl apply -f http://mirror.faasx.com/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```

插件安装完成后，可以通过检查`coredns pod`的运行状态来判断网络插件是否正常运行

```console
root@K8s-1:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY     STATUS    RESTARTS   AGE
kube-system   calico-etcd-nmj5j                         1/1       Running   0          54m
kube-system   calico-kube-controllers-f9d6c4cb6-kxlb2   1/1       Running   0          54m
kube-system   calico-node-85nm9                         2/2       Running   0          13m
kube-system   calico-node-vrhdl                         2/2       Running   0          54m
kube-system   coredns-7997f8864c-2mp87                  1/1       Running   0          59m
kube-system   coredns-7997f8864c-2vl6h                  1/1       Running   0          59m
kube-system   etcd-k8s-1                                1/1       Running   0          58m
kube-system   kube-apiserver-k8s-1                      1/1       Running   0          58m
kube-system   kube-controller-manager-k8s-1             1/1       Running   0          58m
kube-system   kube-proxy-48p2g                          1/1       Running   0          13m
kube-system   kube-proxy-5chhg                          1/1       Running   0          59m
kube-system   kube-scheduler-k8s-1                      1/1       Running   0          58m
```

等待`coredns pod`的状态变成Running，就可以继续添加从节点了。

### 0x05 添加从节点

在从节点上按照前面的步骤按照好docker和kubeadm后，就可以添加从节点到主节点上了

```console
kubeadm join 192.168.5.136:6443 --token dfzpd9.7xxx9kre811fgw1n --discovery-token-ca-cert-hash sha256:31d53cf706073f557b43e5e7acacb47dceb828ccced3c648dbfbfa4d86d74b1c
```

完整输出如下：

```console
root@K8s-2:~# kubeadm join 192.168.5.136:6443 --token dfzpd9.7xxx9kre811fgw1n --discovery-token-ca-cert-hash sha256:31d53cf706073f557b43e5e7acacb47dceb828ccced3c648dbfbfa4d86d74b1c
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[discovery] Trying to connect to API Server "192.168.5.136:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.5.136:6443"
[discovery] Requesting info from "https://192.168.5.136:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.5.136:6443"
[discovery] Successfully established connection with API Server "192.168.5.136:6443"

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

过一会儿就可以通过`kubectl get nodes`命令在主节点上查询到从节点了

```console
root@K8s-1:~# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
k8s-1     Ready     master    1h        v1.10.2
k8s-2     Ready     <none>    1m        v1.10.2
k8s-3     Ready     <none>    23m       v1.10.2
```

### 0x06 卸载集群

想要撤销kubeadm做的事，首先要排除节点，并确保在关闭节点之前要清空节点。

在主节点上运行：

```console
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```

然后在需要移除的节点上，重置kubeadm的安装状态：

```console
kubeadm reset
```

如果你想重新配置集群，只需运行kubeadm init或者kubeadm join并使用所需的参数即可


### 参考资料

+ [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
+ [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
+ [使用kubeadm搭建Kubernetes(1.10.2)集群（国内环境）](https://www.cnblogs.com/RainingNight/p/using-kubeadm-to-create-a-cluster.html)
