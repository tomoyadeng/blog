---
title: kubernetes-installation
tags: kubernetes
date: 2018-05-06 14:32:31
---

### 0x00 前置条件

+ ubuntu16.04 虚拟机
+ shadowsocks 科学上网

### 0x00 准备环境

#### 配置apt源

国内配置apt源为阿里云的源下载速度会快一些，(shadowsocks设置为PAC模式)

```bash
cp -p /etc/apt/sources.list /etc/apt/sources.list.bak

cat <<-EOF | sudo tee /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
EOF
```
