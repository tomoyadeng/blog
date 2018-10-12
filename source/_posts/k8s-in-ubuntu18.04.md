---
title: K8s折腾日记(零) -- 基于 Ubuntu 18.04 安装部署K8s集群
date: 2018-10-12 20:13:48
tags: kubernetes
categories: Cloud Computing
---

__基于 Kubernetes 1.12.1__

之前折腾K8s的时候一直使用的是在 Ubuntu 虚拟机上起的 minikube，最近想在我的笔记本上使用多台虚拟机部署一套 Kubernetes 集群。正好今年上半年 Ubuntu 发布了新的LTS版本 -- 18.04 (Bionic Beaver)，于是便有了这篇文章，在 Ubuntu 18.04 上折腾安装部署 K8s 集群。

<!-- more -->

## 0x00 前置条件

+ Ubuntu 18.04 虚拟机
+ Shadowsocks 科学上网

我的笔记本是Win10，通过 HyperV 安装了 3 台虚拟机，并对虚拟机进行如下规划：

|主机名|ip|角色|配置|OS|
|---|---|---|---|---|
|k8s-master001|192.168.99.103|master|2C2G|Ubuntu 18.04|
|k8s-node001|192.168.99.104|node|1C2G|Ubuntu 18.04|
|k8s-node002|192.168.99.105|node|1C2G|Ubuntu 18.04|

## 0x01 环境准备(所有虚拟机)

### Ubuntu虚拟机配置apt源

国内配置apt源为阿里云的源下载速度会快一些，把 `/etc/apt/sources.list` 里面的内容替换成下面的内容即可

```console
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

更新

```bash
apt-get update
```

### 配置 Shadowsocks 客户端进行科学上网

#### 1. 安装 Shadowsocks 客户端

参考 [Shadowsocks 设置方法 (Linux)](https://github.com/Shadowsocks-Wiki/shadowsocks/blob/master/6-linux-setup-guide-cn.md#%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%AE%A2%E6%88%B7%E7%AB%AF)

```console
apt-get install python-pip
pip install git+https://github.com/shadowsocks/shadowsocks.git@master
```

#### 2. 创建 Shadowsocks 配置文件

创建一个 `/etc/shadowsocks.json` 文件，格式如下

```json
{
    "server":"服务器 IP 或是域名",
    "server_port":端口号,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"密码",
    "timeout":300,
    "method":"加密方式 (chacha20-ietf-poly1305 / aes-256-cfb)",
    "fast_open": false
}
```

#### 3. 启动 Shadowsocks

```console
/usr/local/bin/sslocal -c /etc/shadowsocks.json -d start
```

#### 4. 安裝 privoxy

为了在终端中使用代理，还需要安装配置 privoxy (也可以选择 proxychains)

```console
apt-get install privoxy -y
```

#### 5. 配置 privoxy

安装完成后编辑 `/etc/privoxy/config`，搜索关键字 `forward-socks5t`，取消下面这一行的注释:

```console
forward-socks5t   /               127.0.0.1:1080 .
```

这里 1080 对应着上面 Shadowsocks 配置文件中的“端口号”。

#### 6. 启动 privoxy

```bash
systemctl start privoxy
```

#### 7. 配置 proxy

编辑 `~/.profile`，并在文件末尾添加如下内容：

```console
export http_proxy=http://127.0.0.1:8118
export https_proxy=http://127.0.0.1:8118
export ftp_proxy=http://127.0.0.1:8118
export no_proxy=localhost,127.0.0.0,127.0.1.1,127.0.1.1,10.0.0.0/8,192.168.0.0/16,mirrors.aliyun.com

export HTTP_PROXY=http://127.0.0.1:8118
export HTTPS_PROXY=http://127.0.0.1:8118
export FTP_PROXY=http://127.0.0.1:8118
export NO_PROXY=localhost,127.0.0.0,127.0.1.1,127.0.1.1,10.0.0.0/8,192.168.0.0/16,mirrors.aliyun.com
```

执行 `source ~/.profile` 使之生效。

测试proxy是否可用

```console
curl www.google.com
```

### 关闭swap

编辑`/etc/fstab`文件，注释掉引用swap的行，保存并重启后输入`sudo swapoff -a`即可。参考[Kubelet/Kubernetes should work with Swap Enabled](https://github.com/kubernetes/kubernetes/issues/53533)

## 0x02 安装Docker(所有虚拟机)

### 安装

参考 [安装Docker-阿里云](https://www.alibabacloud.com/help/zh/doc-detail/60742.htm)

```bash
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并指定版本安装Docker-CE （这里安装 18.06.1~ce~3-0~ubuntu）
sudo apt-get -y update
apt-cache madison docker-ce

sudo apt-get -y install docker-ce=18.06.1~ce~3-0~ubuntu
```

### 配置阿里云镜像加速

参考 [Docker 镜像加速器](https://yq.aliyun.com/articles/29941)

配置加速地址，并重启 Docker

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://[阿里云分配的私有地址].mirror.aliyuncs.com"]
}
EOF
```

### 配置 Docker 的 proxy

Kubernetes 的一些 `docker` 镜像是需要借助梯子才能拉取到的，为此需要为 `Docker` 配置 Proxy。

参考 [https://docs.docker.com/config/daemon/systemd/#httphttps-proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)

配置文件 `/etc/systemd/system/docker.service.d/http-proxy.conf` 内容如下：

```conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8118/"
Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,192.168.0.0/16,[阿里云分配的私有地址].mirror.aliyuncs.com"
```

注意：这里的NO_PROXY需要上面配置的镜像加速地址添加进去

重启 Docker

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 0x03 安装Kubernetes(所有虚拟机)

参考 [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

注意： 目前kubernetes还没有 Ubuntu18.04 的编好的版本，用的 16.04 xenial 的二进制文件

## 0x04 配置master节点

### 使用kubeadm init初始化master节点

参考 [https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

```console
kubeadm init --apiserver-advertise-address 192.168.99.103 --pod-network-cidr=10.244.0.0/16
```

init 常用主要参数：

+ --kubernetes-version: 指定Kubenetes版本，如果不指定该参数，会从google网站下载最新的版本信息。
+ --pod-network-cidr: 指定pod网络的IP地址范围，它的值取决于你在下一步选择的哪个网络网络插件，比如我在本文中使用的是 flannel 网络，需要指定为10.244.0.0/16。
+ --apiserver-advertise-address: 指定master服务发布的Ip地址，如果不指定，则会自动检测网络接口，通常是内网IP。

kubeadm init 输出的token用于master和加入节点间的身份认证，token是机密的，需要保证它的安全，因为拥有此标记的人都可以随意向集群中添加节点。

### 安装网络插件

安装一个网络插件是必须的，因为你的pods之间需要彼此通信。

网络部署必须是优先于任何应用的部署，详细的网络列表可参考[插件页面](https://kubernetes.io/docs/concepts/cluster-administration/addons/)。

本文使用的是[flannel 网络](https://github.com/coreos/flannel)，安装如下：

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

插件安装完成后，可以通过检查`coredns pod`的运行状态来判断网络插件是否正常运行。等待`coredns pod`的状态变成Running，就可以继续添加从节点了。

## 0x05 添加从节点

在从节点上按照前面的步骤按照好docker和kubeadm后，就可以添加从节点到主节点上了

```console
kubeadm join 192.168.99.103:6443 --token k3c625.qjgzf8bdc8naufzt --discovery-token-ca-cert-hash sha256:3edad91bb59c6657e4b3dea984e9f484c56032b0d81d504e7e6e3615072f334b
```

过一会儿就可以通过 `kubectl get nodes` 命令在主节点上查询到从节点了

```console
root@k8s-master001:~/k8s# kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
k8s-master001   Ready    master   21h   v1.12.1
k8s-node001     Ready    <none>   21h   v1.12.1
k8s-node002     Ready    <none>   21h   v1.12.1
```

## 0x06 安装 Dashboard 插件

dashboard 支持两种模式：

__https__

必须提供证书，参考 [官方部署文档](https://github.com/kubernetes/dashboard/wiki/Installation#recommended-setup)

    kubectl apply -f https://github.com/kubernetes/dashboard/blob/master/src/deploy/recommended/kubernetes-dashboard.yaml

__http__

    kubectl apply -f https://github.com/kubernetes/dashboard/blob/master/src/deploy/alternative/kubernetes-dashboard.yaml

我这里先把配置文件下到本地：

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard.yaml
```

并在配置文件最后添加 `type: NodePort` 以实现外网访问

```yaml
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
```

执行 `kubectl create -f kubernetes-dashboard.yaml` 安装。

为了简单，我这里直接为 Dashboard 赋予 Admin 的权限，参考[https://github.com/kubernetes/dashboard/wiki/Access-control#admin-privileges](https://github.com/kubernetes/dashboard/wiki/Access-control#admin-privileges)

新增 `dashboard-admin.yaml` 并使用 `kubectl create -f dashboard-admin.yaml` 进行部署，文件内容如下：

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

接下来就可以直接通过 nodeIp:port 访问 Dashboard 了。暴露的外部端口可以通过 `kubectl get svc --namespace=kube-system` 查看

```console
root@k8s-master001:~/k8s# kubectl get svc --namespace=kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   21h
kubernetes-dashboard   NodePort    10.99.58.148   <none>        80:32094/TCP    20h
```

## 0x07 卸载集群

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
