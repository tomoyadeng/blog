---
title: spring-boot-mybatis-docker整合使用
date: 2017-07-23 17:13:17
tags: spring boot
---

### 0x00 前置条件
1. [Java](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
2. [Maven](https://maven.apache.org/download.cgi)
2. [Docker](https://docs.docker.com/engine/installation/), [Docker Compose](https://docs.docker.com/compose/install/)

### 0x01 使用maven新建Spring Boot工程 

按工程根目录的相对路径创建如下文件

`pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.tomoyadeng</groupId>
    <artifactId>demo-springboot-docker</artifactId>
    <version>1.0.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

<!-- more -->

`src/main/java/demo/Application.java`
```java
package demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class Application {
    @RequestMapping("/")
    public String home() {
        return "Get started";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

这样，一个简单的Spring boot的应用就创建OK。可使用`mvn package`编译打包为jar， 然后使用命令行`java -jar target/demo-springboot-docker-1.0.0.jar`直接启动

### 0x02 将应用docker化

首先创建应用的Dockerfile

`src/main/docker/Dockerfile`
```dockerfile
FROM java:8
VOLUME /tmp
ADD demo-springboot-docker-1.0.0.jar app.jar
RUN sh -c 'touch /app.jar'
ENV JAVA_OPTS=""
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
```

然后在pom.xml中添加maven插件依赖，以支持构建docker镜像

`pom.xml`

```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.4.11</version>
    <configuration>
        <imageName>tomoyadeng/${project.artifactId}</imageName>
        <dockerDirectory>src/main/docker</dockerDirectory>
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <directory>${project.build.directory}</directory>
                <include>${project.build.finalName}.jar</include>
            </resource>
        </resources>
    </configuration>
</plugin>
```

使用`mvn install docker:build`即可构建docker镜像，构建完成后，`docker images`可查看当前的镜像。

`docker run -p 8080:8080 -t tomoyadeng/demo-springboot-docker` 可以启动docker容器，此时就完成了此应用的docker化
```bash
tomoya@ubuntu:~/Code/demo-springboot-docker$ docker run -p 8080:8080 -t tomoyadeng/demo-springboot-docker

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.4.RELEASE)
2017-07-23 11:53:34.894  INFO 5 --- [           main] demo.Application                         : Starting Application v1.0.0 on f1ff304f4b94 with PID 5 (/app.jar started by root in /)
...
```

可使用`docker stop`和`docker rm`来停止容器运行

```bash
tomoya@ubuntu:~/Code/demo-springboot-docker$ docker ps
CONTAINER ID        IMAGE                               COMMAND                  CREATED              STATUS              PORTS                    NAMES
f1ff304f4b94        tomoyadeng/demo-springboot-docker   "sh -c 'java $JAVA..."   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   keen_swartz
$ docker stop f1ff304f4b94
$ docker rm f1ff304f4b94
```

### 0x03 创建MyBatis的demo

首先，在`pom.xml`中添加mybatis和mysql-connector的依赖

`pom.xml`
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.0</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>6.0.6</version>
</dependency>
```

添加object类，此处省略了getter和setter

`src/main/java/demo/domain/User.java`
```java
package demo.domain;

import java.io.Serializable;

public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private int id;

    private String name;

    private String phoneNo;

    private String email;
}
```

添加mapper

`src/main/java/demo/mapper/UserMapper.java`
```java
package demo.mapper;

import demo.domain.User;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

@Mapper
public interface UserMapper {
    @Select("select * from tbl_user where name = #{name}")
    User findByName(@Param("name") String name);
}

```

修改Application.java，添加查询数据库的操作

`src/main/java/demo/Application.java`
```java
package demo;

import demo.domain.User;
import demo.mapper.UserMapper;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import static org.springframework.web.bind.annotation.RequestMethod.GET;

@SpringBootApplication
@RestController
public class Application {
    final private UserMapper userMapper;

    public Application(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    @RequestMapping("/")
    public String home() {
        return "Get started";
    }

    @RequestMapping(value = "/user", method = GET)
    public String getUserByName(@RequestParam("name") String name) {
        User user = this.userMapper.findByName(name);
        return user == null ? "No such user!" : user.toString();
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

添加application.yml配置文件

`src/main/resources/application.yml`
```yml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mybatis?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC&useSSL=false
    driverClassName: com.mysql.cj.jdbc.Driver
    username: root
    password: 123456
    schema: classpath:schema.sql
---
spring:
  profiles: container
  datasource:
    url: jdbc:mysql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC&useSSL=false
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
    schema: classpath:schema.sql
    initialize: true
```

附：schema.sql

`src/main/resources/schema.sql`
```sql
drop table if exists tbl_user;

create table tbl_user(id int primary key auto_increment,name varchar(32),phoneNo varchar(16), email varchar(32));

insert into tbl_user(name, phoneNo, email) values ('dave', '13012345678', 'dave@tomoyadeng.com');
```

修改Dockerfile，主要是修改ENTRYPOINT

`src/main/docker/Dockerfile`
```dockerfile
FROM java:8
VOLUME /tmp
ADD demo-springboot-docker-1.0.0.jar app.jar
RUN sh -c 'touch /app.jar'
ENV JAVA_OPTS=""
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-Dspring.profiles.active=container","-jar","/app.jar"]
```


### 0x04 手动启动docker应用

首先，我们需要先启动一个mysql的容器，执行下面命令即可

```bash
docker run -d \
    --name mybatis-mysql \
    -e MYSQL_ROOT_PASSWORD=123456 \
    -e MYSQL_DATABASE=mybatis \
    -e MYSQL_USER=dbuser \
    -e MYSQL_PASSWORD=123456 \
    mysql:latest
```

启动完成后，可用`docker ps`查看，也可以通过执行下面命令连接到mysql

```bash
docker run -it --link mybatis-mysql:mysql --rm mysql sh \
    -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```

然后，启动应用容器并连接到mysql

```bash
docker run -d -t \
    --name demo-springboot-docker \
    --link mybatis-mysql:mysql \
    -p 8088:8080 \
    -e DATABASE_HOST=mybatis-mysql \
    -e DATABASE_PORT=3306 \
    -e DATABASE_NAME=mybatis \
    -e DATABASE_USER=root \
    -e DATABASE_PASSWORD=123456 \
    tomoyadeng/demo-springboot-docker
```

启动完成后，使用`docker ps`查看，或者直接访问url测试

```bash
curl http://localhost:8088/user?name=dave
```

### 0x05 使用docker-compose启动

在项目根路径下增加docker-compose的配置文件

`docker-compose.yml`

```yaml
version: '3.3'

services:
  mybatis-mysql:
    image: mysql:latest
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=mybatis
      - MYSQL_USER=dbuser
      - MYSQL_PASSWORD=123456
  demo-springboot-docker:
    image: tomoyadeng/demo-springboot-docker
    depends_on:
      - mybatis-mysql
    ports:
      - 8088:8080
    environment:
      - DATABASE_HOST=mybatis-mysql
      - DATABASE_USER=root
      - DATABASE_PASSWORD=123456
      - DATABASE_NAME=mybatis
      - DATABASE_PORT=3306
```

启动前，先将之前手动启动的容器停掉

```bash
docker stop demo-springboot-docker
docker stop mybatis-mysql
docker rm demo-springboot-docker
docker rm mybatis-mysql
```

然后直接使用命令启动

```bash
docker-compose up
```
