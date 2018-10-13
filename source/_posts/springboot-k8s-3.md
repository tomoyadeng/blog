---
title: K8s折腾日记(三)--为MySQL提供REST API
date: 2018-09-23 11:10:13
tags:
  - kubernetes
  - mysql
categories: Cloud Computing
---

在上一次的日志中，已经在 Kubernetes 中部署了一个单实例的 MySQL，接下来就是创建一个微服务来提供 MySQL 的 REST API， 让其他微服务能够通过 REST API 来访问数据库中的资源。

<!-- more -->

[本项目源码](https://github.com/tomoyadeng/demo-springboot-k8s)

## 0x01 新建 Spring boot 服务

在原有的父工程下新建一个 Module 命名为 `k8s-db`，新增如下的依赖

```groovy
compile('org.springframework.boot:spring-boot-starter-web')
compile("org.springframework.boot:spring-boot-starter-data-jpa")
compile 'mysql:mysql-connector-java'
compile group: 'javax.xml.bind', name: 'jaxb-api', version: '2.3.0'
```

注： `k8s-db` 是 `demo-springboot-k8s` 下的子工程，会继承在父工程引入的一些依赖包和定义的 task。`jaxb-api` 是在使用 JDK 9 以上的 JDK 版本时需要引入的依赖，因为它是 Java EE 里面的 module，在JDK 9 之后便从默认的包里面移除了，需要手动引入。

### 使用 Spring JPA 进行数据访问

1. 创建一个 Application class

    新建文件 `com/tomoyadeng/demo/springboot/k8s/db/K8sDbApplication.java`

    ```java
    package com.tomoyadeng.demo.springboot.k8s.db;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;

    @SpringBootApplication
    public class K8sDbApplication {
        public static void main(String[] args) {
            SpringApplication.run(K8sDbApplication.class, args);
        }
    }
    ```

2. 定义一个简单的 Entity

    新建文件 `com/tomoyadeng/demo/springboot/k8s/db/domain/Person.java`

    ```java
    package com.tomoyadeng.demo.springboot.k8s.db.domain;

    import lombok.Data;

    import javax.persistence.Entity;
    import javax.persistence.GeneratedValue;
    import javax.persistence.GenerationType;
    import javax.persistence.Id;

    @Data
    @Entity
    public class Person {
        @Id
        @GeneratedValue(strategy = GenerationType.AUTO)
        private long id;

        private String firstName;
        private String lastName;
    }
    ```

3. 创建 JPA repository 提供简单查询

    新建文件 `com/tomoyadeng/demo/springboot/k8s/db/repository/PersonRepository.java`

    ```java
    package com.tomoyadeng.demo.springboot.k8s.db.repository;

    import com.tomoyadeng.demo.springboot.k8s.db.domain.Person;
    import org.springframework.data.repository.CrudRepository;

    import java.util.List;

    public interface PersonRepository extends CrudRepository<Person, Long> {
        List<Person> findByLastName(String lastName);
    }
    ```

上面这几步操作是使用 Spring JPA 进行数据访问的步骤，参考 [Accessing Data with JPA](https://spring.io/guides/gs/accessing-data-jpa/)

### 创建 RestController

新建文件 `com/tomoyadeng/demo/springboot/k8s/db/controller/PersonController.java`

```java
package com.tomoyadeng.demo.springboot.k8s.db.controller;

import com.tomoyadeng.demo.springboot.k8s.db.domain.Person;
import com.tomoyadeng.demo.springboot.k8s.db.repository.PersonRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.util.Assert;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping(path="/person")
public class PersonController {
    @Autowired
    private PersonRepository personRepository;

    @GetMapping("")
    public @ResponseBody List<Person> findAll() {
        List<Person> results = new ArrayList<>();
        personRepository.findAll().forEach(results::add);
        return results;
    }

    @PostMapping("")
    public @ResponseBody Person add(@RequestBody Person person) {
        return personRepository.save(person);
    }

    @PutMapping("/{id}")
    public @ResponseBody Person update(@PathVariable("id") Long id,
            @RequestBody Person person) {
        Assert.isTrue(id == person.getId(), "id not match");
        return personRepository.save(person);
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable("id") Long id) {
        personRepository.deleteById(id);
    }
}
```

## 0x02 本地测试

经过上面的步骤，一个使用 JPA 访问数据库的 Spring boot 的简单微服务就基本算是创建完了，下面需要在本地进行测试，看 REST API 能否正常访问。

使用 Sping profiles 来定义不同运行环境下的环境变量以及其他参数，我这里本地开发环境(Win10)定义为 `dev`，默认便使用 `dev`。

新建 `application.yaml`

```yaml
spring:
  profiles:
    active: dev
```

新建 `application-dev.yaml`

```yaml
spring:
  profiles: dev
  jpa:
    hibernate:
      ddl-auto: update
  datasource:
    url: jdbc:mysql://localhost:3306/k8s?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC&useSSL=false
    driverClassName: com.mysql.jdbc.Driver
    username: root
    password: 123456
```

需要提前在本地安装好 MySQL， 并新建一个名为 k8s 的数据库。

Spring boot 应用启动后，URL： `http://localhost:8080/person` 能正常访问即可。

## 0x03 构建 Docker 镜像

要使这个 Spring boot 应用跑在 Kubernetes 上，还是照例要先把它构建成 Docker 镜像。首先，要先新建一个 k8s 的 Spring profile。

新建文件： `application-k8s.yaml`

```yaml
spring:
  profiles: k8s
  jpa:
    hibernate:
      ddl-auto: update
  datasource:
    url: jdbc:mysql://${MYSQL_HOST}:${MYSQL_PORT}/k8s?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC&useSSL=false
    driverClassName: com.mysql.jdbc.Driver
    username: ${MYSQL_USER}
    password: ${MYQL_PASSWD}
```

MYSQL_HOST 等变量均需要在启动 docker 容器时注入。

随后便可以构建一个 docker 镜像了，我这里使用 gradle 来直接构建。

```bash
gradle build buildDocker
```

## 0x04 部署 k8s-db 微服务到 Kubernetes

### 新建 k8s 数据库

在部署 k8s-db 微服务之前，需要先在 MySQL 中创建使用到的db，使用 `kubectl exec` 命令进入已经启动的容器的bash

```bash
kubectl exec -it mysql-client-585479c646-nzl8m -- mysql -h mysql -p123456
```

```sql
create database k8s;
```

这样，数据库就创建好了，接下来部署 k8s-db 微服务。

### 部署

新建文件 `k8s-db.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-db
  labels:
    app: k8s-db
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: k8s-db
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-db
  labels:
    app: k8s-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-db
  template:
    metadata:
      labels:
        app: k8s-db
    spec:
      containers:
        - name: k8s-db
          image: com.tomoyadeng/k8s-db:1.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          env:
            - name: MYSQL_HOST
              value: mysql
            - name: MYSQL_PORT
              value: '3306'
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: username
            - name: MYQL_PASSWD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
```

通过 `kubectl create -f k8s-db.yaml` 部署：

```console
root@k8s:~/k8s# kubectl create -f k8s-db.yaml
service/k8s-db created
deployment.apps/k8s-db created
```

随后便可以通过 `kubectl get pod` 看到新启动的 pod 了。

## 0x05 参考资料

+ [Accessing Data with JPA](https://spring.io/guides/gs/accessing-data-jpa/)
+ [在 Kubernetes 上构建和部署 Java Spring Boot 微服务](https://github.com/IBM/spring-boot-microservices-on-kubernetes/blob/master/README-cn.md)
