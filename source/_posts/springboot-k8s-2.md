---
title: K8s折腾日记(二)--在Kubernetes中部署一个单实例有状态的应用
tags:
  - kubernetes
  - mysql
categories: Cloud Computing
date: 2018-09-16 11:52:22
---

之前已经尝试在Kubernetes中部署了一个简单的Spring boot应用，但这个应用是无状态的，这次主要折腾一下如何运行一个单实例有状态应用(MySQL)

<!-- more -->

## 0x01 定义持久化磁盘

Kubernetes 中使用 Volume 来持久化保存容器的数据，Volume 的生命周期独立于容器， Pod 中的容器可能频繁的被销毁和重建，但 Volume 会被保留。

从本质上来将， Kubernetes Volume 是一个目录，但 Volume 提供了对各种backend 的抽象，容器使用 Volume 时不用关心数据到底是存在本地节点的文件系统中还是类似 EBS 这种云硬盘中。

接下来，就通过 [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 定义一个持久化磁盘：

为了方便，使用 hostPath 这种 Volume，这种类型是映射到主机上目录的卷，应该只用于测试目的或者单节点集群。

新建 `local-volume.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mysql
```

创建持久卷：

```bash
kubectl create -f local-volume.yaml
```

随后就能通过 `kubectl get pv` 看到刚创建好的 PV 了。

## 0x02 创建 MySQL 密码 Secret

在 Kubernetes 中可以通过 [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) 对象来保存类似于数据库用户名、密码或者密钥等敏感信息。Secret 会以密文的方式存储数据，避免了直接在配置文件中保存敏感信息，且 Secret 会以 Volume 的形式 mount 到 Pod，这样容器便可以使用这些数据了。

接下来通过yaml文件来创建数据库的 Secret，yaml文件中的敏感数据必须是通过 base64 编码后的结果。

```console
root@k8s:~# echo -n root | base64
cm9vdA==
root@k8s:~# echo -n 123456 | base64
MTIzNDU2
```

新建 `mysql-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  username: cm9vdA==
  password: MTIzNDU2
```

创建 Secret

```
kubectl create -f mysql-secret.yaml
```

随后便能通过 `kubectl get secret` 查看刚才创建的 Secret

## 0x03 部署 MySQL

PersistentVolume (PV) 是外部存储系统的一块存储空间，在 Kubernetes 中，可通过 PersistentVolumeClaim (PVC) 来申请现已存在的 PV 的使用。

接下来通过yaml文件部署 MySQL：

新建 `mysql-deployment.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

部署 MySQL

```bash
kubectl create -f mysql-deployment.yaml
```

随后便能通过 `kubectl get pod` 查看到部署的mysql了。

## 0x04 访问 MySQL 实例

运行MySQL客户端以连接到服务器:

```bash
kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h <pod-ip> -p<password>
```

这个命令在集群内创建一个新的Pod并运行MySQL客户端,并通过服务将其连接到服务器.如果连接成功,就知道有状态的MySQL database正处于运行状态.

    root@k8s:~/k8s# kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h mysql -p123456
    If you don't see a command prompt, try pressing enter.
    mysql>

至此，单实例的 MySQL 就在 Kubernetes 中部署成功了。

## 0x05 参考资料

+ [运行一个单实例有状态应用](https://kubernetes.io/cn/docs/tasks/run-application/run-single-instance-stateful-application/)
+ [基于 Persistent Volumes 搭建 WordPress 和 MySQL 应用](https://kubernetes.io/cn/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)
