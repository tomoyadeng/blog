---
title: K8s上部署MySQL主从
tags:
  - kubernetes
  - mysql
categories: Cloud Computing
date: 2018-10-31 19:34:59
---

K8s实验环境从minikube升级到单主多节点的集群后，接下来就准备把单实例的MySQL升级为MySQL主从架构。本文就介绍下怎么在K8s集群中部署MySQL主从，数据持久化采用nfs。

<!-- more -->

## 0x01 搭建 NFS 服务器

参考 [https://linuxconfig.org/how-to-configure-a-nfs-file-server-on-ubuntu-18-04-bionic-beaver](https://linuxconfig.org/how-to-configure-a-nfs-file-server-on-ubuntu-18-04-bionic-beaver)

```bash
apt install nfs-kernel-server
```

编辑 `/etc/exports`

```console
/nfs		*(rw,sync,no_subtree_check,no_root_squash)
```

重启

```bash
systemctl restart nfs-kernel-server
```

创建目录

```bash
mkdir -p /nfs/mysql/data/master
mkdir -p /nfs/mysql/data/replica
```

## 0x02 准备 MySQL 主从镜像

我们可以通过对MySQL官方的Dockerfile进行修改，然后构建生成主从镜像，首先把官方镜像克隆下来

```bash
git clone https://github.com/docker-library/mysql.git
```

当然也可以直接把 `Dockerfile` 和 `docker-entrypoint.sh` 下载到本地(这里选择MySQL 5.7 版本)

[https://github.com/docker-library/mysql/tree/master/5.7](https://github.com/docker-library/mysql/tree/master/5.7)

### 修改 master Dockerfile

1. 将 `Dockerfile` 和 `docker-entrypoint.sh` 拷贝一份到一个新的目录，用于构建 master 镜像。

2. 在 `Dockerfile` 中 `VOLUME /var/lib/mysql` 这一行前面加一行。这一行的作用是将mysql master的server-id设置为1。

```bash
RUN sed -i '/\[mysqld\]/a server-id=1\nlog-bin' /etc/mysql/mysql.conf.d/mysqld.cnf
```

3. 在docker-entrypoint.sh中添加如下内容，创建一个复制用户并赋权限，刷新系统权限表

```bash
echo "CREATE USER '$MYSQL_REPLICATION_USER'@'%' IDENTIFIED BY '$MYSQL_REPLICATION_PASSWORD' ;" | "${mysql[@]}" 
echo "GRANT REPLICATION SLAVE ON *.* TO '$MYSQL_REPLICATION_USER'@'%' IDENTIFIED BY '$MYSQL_REPLICATION_PASSWORD' ;" | "${mysql[@]}" 
echo 'FLUSH PRIVILEGES ;' | "${mysql[@]}"
```

{% asset_img master-1.png %}

### 修改 replica Dockerfile

1. 将 `Dockerfile` 和 `docker-entrypoint.sh` 拷贝一份到一个新的目录，用于构建 replica 镜像。

2. 在 `Dockerfile` 中 `VOLUME /var/lib/mysql` 这一行前面添加如下内容，将mysql replica的server-id设置为一个随机数:

```bash
RUN RAND="$(date +%s | rev | cut -c 1-2)$(echo ${RANDOM})" && sed -i '/\[mysqld\]/a server-id='$RAND'\nlog-bin' /etc/mysql/mysql.conf.d/mysqld.cnf
```

3. 在docker-entrypoint.sh中添加如下内容，配置连接master主机的host、user、password等参数，并启动复制进程。

```bash
echo "STOP SLAVE;" | "${mysql[@]}" 
echo "CHANGE MASTER TO master_host='$MYSQL_MASTER_SERVICE_HOST', master_user='$MYSQL_REPLICATION_USER', master_password='$MYSQL_REPLICATION_PASSWORD' ;" | "${mysql[@]}" 
echo "START SLAVE;" | "${mysql[@]}"
```

{% asset_img replica-1.png %}

### 构建镜像

```bash
cd ~/dockerfile/mysql/master/
docker build -t tomoyadeng/mysql-master:5.7.1
docker push tomoyadeng/mysql-master:5.7.1

cd ~/dockerfile/mysql/replica01
docker build -t tomoyadeng/mysql-replica:5.7.1
docker push tomoyadeng/mysql-replica:5.7.1
```

构建的时候可能会出错，重试几次就好了，[docker-library/official-images#4252 (comment)](https://github.com/docker-library/official-images/issues/4252#issuecomment-381783035)

也可以选择直接使用我构建好的镜像

```bash
docker pull tomoyadeng/mysql-master:5.7.1
docker pull tomoyadeng/mysql-replica:5.7.1
```

## 0x03 创建 Secret

新建 `mysql-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  rootuser: cm9vdA==
  rootpwd: MTIzNDU2
  repluser: cmVwbA==
  replpwd: MTIzNDU2
```

创建 Secret

```bash
kubectl create -f mysql-secret.yaml
```

## 0x04 创建 PV

新建 `master-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: master-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: master
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/mysql/data/master
    server: 192.168.99.103
  
```

随后通过`kubectl create -f master-pv.yaml` 创建 master pv

新建 `replica-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: replica-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: replica
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/mysql/data/replica
    server: 192.168.99.103
  
```

随后通过`kubectl create -f replica-pv.yaml` 创建 replica pv

## 0x05 部署 master

新建 `mysql-master-deployment.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
  labels:
    app: mysql-master
spec:
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
  selector:
    app: mysql-master
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-master-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: master
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-master
  labels:
    app: mysql-master
spec:
  selector:
    matchLabels:
      app: mysql-master
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql-master
    spec:
      containers:
      - image: tomoyadeng/mysql-master:5.7.1
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: rootpwd
        - name: MYSQL_REPLICATION_USER
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: repluser
        - name: MYSQL_REPLICAITON_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: replpwd
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-master-pvc
```

上面的yaml文件包含了创建 master的 PVC，Deployment 和 Service，直接通过 `kubectl create -f mysql-master-deployment.yaml` 进行部署。

部署完成后，可以通过 `kubectl get svc | grep master` 查看刚才部署的 Service，然后可以通过运行MySQL客户端以连接到服务器

```bash
root@k8s-master001:~/k8s/mysql-cluster# kubectl run -it --rm --image=mysql:5.7 mysql-client -- mysql -h 10.97.247.244 -p123456

kubectl run --generator=deployment/apps.v1beta1 is DEPRECATED and will be removed in a future version. Use kubectl create instead.
If you don't see a command prompt, try pressing enter.

mysql> 
```

这里的ip是通过 `kubectl get svc` 查到的 CLUSTER-IP

## 0x06 部署 replica

新建 `mysql-replica-deployment.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-replica
  labels:
    app: mysql-replica
spec:
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
  selector:
    app: mysql-replica
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-replica-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: replica
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-replica
  labels:
    app: mysql-replica
spec:
  selector:
    matchLabels:
      app: mysql-replica
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql-replica
    spec:
      containers:
      - image: tomoyadeng/mysql-replica:5.7.1
        name: mysql
        env:
        - name: MYSQL_MASTER_SERVICE_HOST
          value: mysql-master
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: rootpwd
        - name: MYSQL_REPLICATION_USER
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: repluser
        - name: MYSQL_REPLICAITON_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: replpwd
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-replica-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-replica-storage
        persistentVolumeClaim:
          claimName: mysql-replica-pvc
```

这里的 `MYSQL_MASTER_SERVICE_HOST` 环境变量配置的就是之前master svc的名称。随后，直接使用 `kubectl create -f mysql-replica-deployment.yaml` 进行部署。过一会儿，可以通过运行客户端连接 replica，查看备份状态

{% asset_img status.png %}

## 0x07 总结

至此，MySQL主从就搭建好了，工程源文件见 [https://github.com/tomoyadeng/demo-springboot-k8s/tree/master/mysql-cluster](https://github.com/tomoyadeng/demo-springboot-k8s/tree/master/mysql-cluster)
