---
title: 使用Docker快速搭建Kafka开发环境
tags: 'kafka, docker'
date: 2018-06-02 17:46:22
---


最近准备学习Kafka，于是买了《Kafka权威指南》来看。作为一个初学者，快速搭建起一套可以运行的环境十分重要，跟着该书第2章的安装介绍可以完成在Linux系统下的环境搭建，但是读下来发现步骤还是有点繁多。有没有什么更加快捷的办法搭建一套可以运行的开发环境呢，于是我想到了Docker。2018年了，容器化已经成为了主流，在本地进行开发和测试的时候使用Docker也便于模拟多节点的集群环境。

### 0x00 前置条件

+ [Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/):  要想使用Docker来启动kafka，开发环境提前装好Docker是必须的，我一般在Ubuntu虚拟机上进行开发测试
+ [Docker Compose](https://docs.docker.com/compose/install/): kafka依赖zookeeper，使用docker-compose来管理容器依赖

### 0x01 Docker镜像

要想使用Docker安装Kafka，第一件事当然是去Docker hub上找镜像以及使用方法啦。发现kafka并不像mysql或者redis那样有官方镜像，不过Google一下后发现可以选择知名的三方镜像[wurstmeister/kafka](https://hub.docker.com/r/wurstmeister/kafka/)

[wurstmeister/kafka](https://github.com/wurstmeister/kafka-docker)在Github上更新还算频繁，目前使用kafka版本是`1.1.0`

### 0x02 安装

1. 参考[官方测试用的docker-compose.yml](https://github.com/wurstmeister/kafka-docker/blob/master/test/docker-compose.yml)直接在自定义的目录位置新建docker-compose的配置文件

    `touch ~/docker/kafka/docker-compose.yml`

    ```yaml
    version: '2.1'
    services:
    zookeeper:
        image: wurstmeister/zookeeper
        ports:
        - "2181"
    kafka:
        image: wurstmeister/kafka
        ports:
        - "9092"
        environment:
        KAFKA_ADVERTISED_HOST_NAME: 192.168.5.139
        KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
        volumes:
        - /var/run/docker.sock:/var/run/docker.sock
    ```

    __注意：__ `KAFKA_ADVERTISED_HOST_NAME` 需要配置为宿主机的ip

2. `docker-compose` 启动kafka

    ```console
    root@ubuntu:~/docker/kafka# docker-compose up -d
    ```

    启动完之后通过`docker ps`可以看到启动了一个zookeeper容器和一个kafka容器

3. 启动多个kafka节点，比如3

    ```console
    root@ubuntu:~/docker/kafka# docker-compose scale kafka=3
    ```

    如果没什么错误的话，再通过`docker ps`可以看到启动了一个zookeeper容器和三个kafka容器

### 0x03 验证

1. 首先进入到一个kafka容器中，例如: kafka_kafka_1

    ```console
    root@ubuntu:~/docker/kafka# docker exec -it kafka_kafka_1 /bin/bash
    ```

2. 创建一个topic并查看，需要指定zookeeper的容器名(这里是kafka_zookeeper_1)，topic的名字为test

    ```console
    $KAFKA_HOME/bin/kafka-topics.sh --create --topic test --zookeeper kafka_zookeeper_1:2181 --replication-factor 1 --partitions 1

    $KAFKA_HOME/bin/kafka-topics.sh --list --zookeeper kafka_zookeeper_1:2181
    ```

3. 发布消息，输入几条消息后，按^C退出发布

    ```console
    $KAFKA_HOME/bin/kafka-console-producer.sh --topic=test --broker-list kafka_kafka_1:9092
    ```

4. 接受消息

    ```console
    $KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server kafka_kafka_1:9092 --from-beginning --topic test
    ```

如果接收到了发布的消息，那么说明部署正常，可以正式使用了。
