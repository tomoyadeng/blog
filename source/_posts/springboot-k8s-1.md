---
title: K8s折腾日记(一)--在Kubernetes中部署spring boot应用
tags:
  - kubernetes
  - spring boot
categories: Cloud Computing
date: 2018-09-13 23:22:43
---


基于容器的微服务架构目前已经成为了开发应用系统的主流，Kubernetes 则是运行微服务应用的理想平台。基于没事儿瞎折腾的态度，自己最近有空闲时间也开始鼓捣起了k8s，经过一步步摸索，终于完成安装和部署。接下来就先分享一下怎么在 kubernetes 中部署一个简单 Spring boot 的应用。

## 0x00 环境准备

+ minikube
+ gradle

minikube 的安装可以参考[使用minikube安装k8s单节点集群](https://yq.aliyun.com/articles/574255)

## 0x01 构建 Spring boot 应用

首先，新建一个Spring Boot应用，姑且命名为`k8s-service`，这个应用就提供一个简单的接口，便于验证是否部署成功。

MainClass:

```java
package com.tomoyadeng.demo.springboot.k8s.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class K8sServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(K8sServiceApplication.class, args);
    }
}
```

<!-- more -->

RestController:

```java
package com.tomoyadeng.demo.springboot.k8s.service.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello, Kubernetes!";
    }
}
```

接下来写个测试用例，在本地跑一下：

```java
package com.tomoyadeng.demo.springboot.k8s.service;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class K8sServiceApplicationTest {

    @Autowired private MockMvc mvc;

    @Test
    public void getHello() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/hello").accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Hello, Kubernetes!")));
    }
}
```

完整代码参考[Github](https://github.com/tomoyadeng/demo-springboot-k8s)

## 0x02 将应用打包成 Docker 镜像

接下来就需要把 Spring boot 应用打包成 docker 镜像了，这样才能在 kubernetes 中运行。将 Spring boot 应用打包成docker镜像可以选择通过 Dockerfile 的方式，或者借助构建工具进行构建。我这里选择借助 gradle 来构建 docker 镜像(使用se.transmode.gradle:gradle-docker:1.2 插件)，在 build.gradle 定义构建docker镜像的任务：

```groovy
docker {
    baseImage "openjdk:8-slim"
    maintainer 'tomoyadeng@gmail.com'
}

task buildDocker(type: Docker, dependsOn: build) {
    applicationName = bootJar.baseName
    tagVersion = bootJar.version
    doFirst {
        copy {
            from bootJar
            into stageDir
        }
    }
    volume "/tmp"
    addFile("${bootJar.baseName}-${bootJar.version}.jar", "app.jar")
    entryPoint(Arrays.asList("java", "-Djava.security.egd=file:/dev/./urandom", "-Dspring.profiles.active=docker", "-jar", "/app.jar"))
}
```

然后执行 `gradle build buildDocker` 就能构建出docker镜像，构建完成后可以通过 `docker images` 查看。

## 0x03 将镜像部署到kubernetes中

### 创建 Deployment

Kubernetes Pod 是由一个或多个容器为了管理和联网的目的而绑定在一起构成的组。Deployment 是管理 Pod 创建和伸缩的推荐方法。要部署刚才构建好的容器镜像，首先要创建 Deployment。

新建 `k8s-service.yaml` 文件

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: k8s-service
  labels:
    app: k8s-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-service
  template:
    metadata:
      labels:
        app: k8s-service
    spec:
      containers:
        - name: k8s-service
          image: com.tomoyadeng/k8s-service:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

运行 `kubectl create -f k8s-service.yaml` 创建 Deployment，创建完成后可以通过 `kubectl get deployments` 查看 deployment，通过 `kubectl get pods` 查看 pod。

### 创建 Service

默认情况下，Pod 只能通过 Kubernetes 集群中的内部 IP 地址访问。要使得 k8s-service 容器可以从 Kubernetes 虚拟网络的外部访问，您必须将 Pod 暴露为 Kubernetes Service。

```console
kubectl expose deployment k8s-service --type=LoadBalancer
```

随后可以通过 `curl $(minikube service k8s-service --url)/hello` 验证部署是否成功。部署的pod的日志可以通过 `kubectl log <Pod Name>` 来查看。

完整代码参考[Github](https://github.com/tomoyadeng/demo-springboot-k8s)
