---
title: 构建 gitbook 的 docker 镜像
date: 2018-12-08 23:46:32
tags: 
    - docker
categories: Tools
---

最近由于更换了VPS服务器，需要把原来老的服务器上发布的 gitbook 迁移到新的服务器上来。在新的服务器上重新安装 gitbook 的时候，遇到了各种恼人的npm/node版本的问题，遂决定构建一个 gitbook 的 docker 镜像，来避免后续在 gitbook 环境和版本问题上浪费时间。

<!-- more -->

## 0x00 构建 gitbook 的 docker 镜像

构建一个docker镜像十分简单，总共就三步：

1. 编写 Dockerfile
2. build
3. push 到 docker hub

### 编写 Dockerfile

gitbook 的依赖很简单，使用 `node` 作为基础镜像，使用 npm 安装`gitbook-cli` 就行：

```dockerfile
FROM node:8.5-alpine
MAINTAINER Tomoya <tomoyadeng@gmail.com>

RUN npm install gitbook-cli -g

ARG GITBOOK_VERSION=3.2.3
RUN gitbook fetch $GITBOOK_VERSION

ENV BOOKDIR /gitbook

VOLUME $BOOKDIR

EXPOSE 4000

WORKDIR $BOOKDIR

CMD ["gitbook", "--help"]
```

### 构建

写好 Dockerfile 后， 直接运行 docker 命令行构建

```bash
docker build . -t tomoyadeng/gitbook:latest
```

这里直接构建的镜像打上了`tomoyadeng/gitbook:latest`的标签，[tomoyadeng](https://hub.docker.com/r/tomoyadeng/) 是我在[Docker Hub](https://hub.docker.com/) 上注册好了的命名空间

### push

随后可以将构建好的镜像推送到 Docker Hub

```bash
docker login

docker push tomoyadeng/gitbook:latest
```

### 使用

```
docker run --rm -v "$PWD:/gitbook" -p 4000:4000 tomoyadeng/gitbook gitbook <command>
```

#### Frequently used gitbook commands

__install__

```
docker run --rm -v "$PWD:/gitbook" -p 4000:4000 tomoyadeng/gitbook gitbook install
```

__build__

```
docker run --rm -v "$PWD:/gitbook" -p 4000:4000 tomoyadeng/gitbook gitbook build
```

__serve__

```
docker run --rm -v "$PWD:/gitbook" -p 4000:4000 tomoyadeng/gitbook gitbook serve
```

more commands or details please refer to [gitbook-cli](https://github.com/GitbookIO/gitbook-cli)

## 0x01 参考资料

+ [gitbook-cli](https://github.com/GitbookIO/gitbook-cli)
