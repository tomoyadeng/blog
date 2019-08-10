---
title: K8s折腾日记 -- 搭建HA集群(1.15.1)
tags: kubernetes
categories: Cloud Computing
date: 2019-08-10 17:58:50
---


貌似从 1.9 版本开始，Kubernetes 就支持通过kubeadm直接起一个Master高可用的集群了，可谓是越来越方便了。正好最近在家里配置了个高性能的台式机，可以起好几台虚拟机了，索性大方点，整个三主三从的HA集群。

官方文档给出了两种HA的拓扑结构，其实就是Etcd在集群内和Etcd在集群外的区别，为了方便，我就直接采用内置Etcd的方式。

+ 内置Etcd

{% asset_img kubeadm-ha-topology-stacked-etcd.svg %}

+ 外置Etcd

{% asset_img kubeadm-ha-topology-external-etcd.svg %}

从拓扑图中可以看出，我们需要一个的 Load Balancer 来实现ApiServer的高可用和负载均衡，所有访问ApiServer的请求都通过 Load Balancer 的 VIP。如果机器在云端，直接购买云服务商提供的负载均衡服务就行了。比如华为云的ELB，以及阿里云的SLB。

由于我是本地起的虚拟机，就不得不自己想办法搭一个 Load Balancer 出来，在网上搜了一下，决定用 Keepalived + haproxy 来搞。

### 0x00 规划

|主机名|ip|角色|配置|OS|
|---|---|---|---|---|
||192.168.137.100|VIP|||
|node-1|192.168.137.101|master|2C4G|Ubuntu 18.04|
|node-2|192.168.137.102|master|2C4G|Ubuntu 18.04|
|node-3|192.168.137.103|master|2C4G|Ubuntu 18.04|
|node-4|192.168.137.104|node|2C4G|Ubuntu 18.04|
|node-5|192.168.137.105|node|2C4G|Ubuntu 18.04|
|node-6|192.168.137.106|node|2C4G|Ubuntu 18.04|

省略 Docker 和 kubernetes 安装的过程，直接按照官网的步骤搞就好了。

这么多台机器都要执行同样的安装操作，想想都有点累，这里给出两个解决办法：

+ 如果是用的虚拟机，应该是有复制虚拟机的功能的，比如我使用的Hyper-V的导出导入功能。先在一台机器上安装好这些基础设施后，再复制成多台，改下IP和Hostname就行了
+ 直接祭出大杀器Ansible，在网上找一个安装Kubernetes的palybook自己改一改，或者自己从头撸一个也是可以的

### 0x01 安装 keepalived

在所有 master 节点安装和配置 keepalived， 注意 state 配置差别

```shell
cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_forward = 1
EOF

sysctl -p

apt install -y keepalived

# 下面的 state 第一台配置为 MASTER 剩下两台配置为 BACKUP
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 250
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 35f18af7190d51c9f7f78f37300a0cbd
    }
    virtual_ipaddress {
        192.168.137.100
    }
    track_script {
        check_haproxy
    }
}
EOF

service keepalived start
service keepalived status

```

### 0x02 安装 HAProxy

在所有 master 节点安装和配置 HAProxy，所有节点配置都是一样的

```shell
cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_nonlocal_bind = 1
EOF

sysctl -p

apt install -y haproxy

# 这里配置监听 VIP 192.168.137.100 的 16443 端口，并将请求转发到后端三台master的6443端口
cat > /etc/haproxy/haproxy.cfg << EOF
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode                    tcp
    log                     global
    option                  redispatch
    retries                 3


listen https-apiserver
    bind 192.168.137.100:16443
    mode        tcp
    balance     roundrobin
    timeout server 15s
    timeout connect 15s
    server  node-1 192.168.137.101:6443 check port 6443 inter 5000 fall 5
    server  node-2 192.168.137.102:6443 check port 6443 inter 5000 fall 5
    server  node-3 192.168.137.103:6443 check port 6443 inter 5000 fall 5

listen http-apiserver
    bind 192.168.137.100:1080
    mode tcp
    balance roundrobin
    timeout server 15s
    timeout connect 15s

    server  node-1 192.168.137.101:8080 check port 6443 inter 5000 fall 5
    server  node-2 192.168.137.102:8080 check port 6443 inter 5000 fall 5
    server  node-3 192.168.137.103:8080 check port 6443 inter 5000 fall 5

EOF


service haproxy start

service haproxy status
```

### 0x03 kubeadm 创建HA集群

```shell
# 确保关闭了swap
swapoff -a

# 假装我有一个域名 cluster.kube.local
cat >> /etc/hosts << EOF
192.168.137.100 cluster.kube.local

192.168.137.101 node-1
192.168.137.102 node-2
192.168.137.103 node-3
192.168.137.104 node-4
192.168.137.105 node-5
192.168.137.106 node-6
EOF

# 启动 HA 集群的 config 文件
# 我准备使用 flannel 网络插件，所以 podSubnet 配置 "10.244.0.0/16"
# 如果是其他网络插件，则这里的 podSubnet 要按要求具体配置
cat > kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.15.1
apiServer:
  certSANs:
    - "cluster.kube.local"
controlPlaneEndpoint: "cluster.kube.local:16443"
networking:
  podSubnet: "10.244.0.0/16"
EOF

# 使用kubeadm init启动HA集群
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
```

顺利的话，会给出两条`kubeadm join`命令，上面的是添加master的命令，下面的添加node的命令。

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    You can now join any number of the control-plane node running the following command on each as root:

    kubeadm join cluster.kube.local:16443 --token 1qjsgg.6s83tvxoo3wma1ps \
        --discovery-token-ca-cert-hash sha256:b454bece5e0292c52794a35aa41b60184d1aad64a3519786e80966fee498d0f5 \
        --control-plane --certificate-key 73b5c18b703b93fde588fece41f1eef7f6ddd693a6ca01067f762f35992afd13

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
    "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join cluster.kube.local:16443 --token 1qjsgg.6s83tvxoo3wma1ps \
        --discovery-token-ca-cert-hash sha256:b454bece5e0292c52794a35aa41b60184d1aad64a3519786e80966fee498d0f5


安装完选择的网络插件(比如 flannel )之后，就可以在其他节点执行对应的命令，添加其他的两个master和三个node了。