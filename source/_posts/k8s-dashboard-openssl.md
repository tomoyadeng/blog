---
title: K8s折腾日记 -- 部署 Dashboard 和 metrics-server
tags: kubernetes
categories: Cloud Computing
date: 2019-08-11 09:54:33
---


Kubernetes Dashboard 从v2.0.0-beta1版本开始，集成了一个metrics-scraper的组件，可以通过 Kubernetes 的 Metrics API 收集一些基础资源的监控信息，并在web页面展示。之前部署dashboard的时候，偷懒使用了http的方式访问，这次决定使用openssl自签发证书来搞。

### 0x01 使用 openssl 签发证书

```shell script
mkdir certs
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj "/C=/ST=/L=/O=/OU=/CN=kubernetes-dashboard"
openssl x509 -req -sha256 -days 10000 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt
```

### 0x02 安装 Dashboard

1. 创建 namespace

```
kubectl create namespace kubernetes-dashboard
```

2. 导入证书

```shell script
kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kubernetes-dashboard
```

3. 部署 dashboard 

```shell script
# 下载部署配置文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta3/aio/deploy/recommended.yaml
```

因为前面自己创建了 kubernetes-dashboard 的 namespace ，这里注释掉 `recommended.yaml` namespace 定义


```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard
```

因为要使用自签发的证书，所以注释掉 kubernetes-dashboard-certs 的 Secret 定义

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque
```

[可选操作] 因为后面想通过 token 的方式登录 dashboard，又不想 token 很快过期，可以通过添加启动参数的方式(`--token-ttl=604800`)来修改 token 的 ttl

```yaml
args:
- --auto-generate-certificates
- --namespace=kubernetes-dashboard
- --token-ttl=604800
```

部署

```shell script
kubectl create -f recommended.yaml
```

### 0x03 使用 NodePort 暴露服务

新建 `external-https-svc.yaml`

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-external
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31443
  selector:
    k8s-app: kubernetes-dashboard
```

部署

```shell
kubectl create -f external-https-svc.yaml
```

随后可以通过 `kubectl -n=kubernetes-dashboard get svc` 查到 service 的端口

### 0x04 serviceAcount 和 token

新建 `admin-user.yaml` (这里使用集群管理员的权限)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

创建用户

`kubectl create -f admin-user.yaml`

查询 token

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

### 0x05 安装 metrics-server

metrics-server 是 Kubernetes 官方的集群资源利用率信息收集器，是 Heapster 瘦身后的替代品。metrics-server 收集的是集群内由各个节点上kubelet暴露出来的利用率信息，算是集群中基础的监控信息了，主要是提供给例如调度逻辑等核心系统使用。

部署过程参考 [metrics-server](https://github.com/kubernetes-incubator/metrics-server)

先从github把代码clone下来

```
git clone https://github.com/kubernetes-incubator/metrics-server.git
```

直接部署

```
cd metrics-server/
kubectl create -f deploy/1.8+/
```

部署完成后需要添加一些启动参数

```
# edit metric-server deployment to add the flags
# args:
# - --kubelet-insecure-tls
# - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
$ kubectl edit deploy -n kube-system metrics-server
```

等 `kubectl top node` 可用，就说明部署好了。

通过浏览器访问 Dashboard，也可以看到 pod 简单的CPU和内存利用率情况。

{% asset_img dashboard-view.png %}

最后，附上一张 [Kubernetes 监控系统的架构图](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/monitoring_architecture.md)

{% asset_img monitoring_architecture.png %}