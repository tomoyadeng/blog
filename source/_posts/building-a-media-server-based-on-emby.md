---
title: 基于Emby搭建媒体服务器
tags:
  - docker
categories: Tools
date: 2019-03-12 22:50:23
---

最近，团队里在开展针对新员工的一系列培训，并将培训过程录制成了一个个的视频。可是分享培训视频却十分的麻烦，基于samba将文件夹共享给团队成员，然后想看培训视频的同学将视频拷贝到本地进行观看，这样实在是太浪费时间和空间。于是，我想到完全可以利用组里的服务器自己搭建一个媒体服务器，把URL共享出去，这样组里面的小伙伴就可以在线观看培训视频了。

### 0x01 选型

有了这个想法，接下来就是去google上找有没有方便快捷的开源项目可以直接使用，寻找方向就是能开箱即用快速搭建，并且在[docker hub](https://hub.docker.com/)上有现成的docker镜像。刚开始我看有人推荐了[jellyfin](https://github.com/jellyfin/jellyfin)，于是赶紧去[docker hub Jellyfin](https://hub.docker.com/r/jellyfin/jellyfin)查一查怎么使用，简单写了个docker-compose文件就将jellyfin起了起来。但在使用的过程中发现jellyfin的性能不太好，我通过浏览器访问时，页面要刷半天才能刷出来。我想着急性子的小伙伴们肯定忍受不了这样的慢网页呀，所以又接着尝试了[Emby](https://emby.media/docker-server.html)，简单测试了下，性能还不错，于是便选定了使用Emby来实现团队的媒体服务器。

### 0x02 快速搭建

有了docker之后，要使用一个开源工具简直不要太方便，总共就三步：

1. 在 docker hub 上找到这个工具的页面
2. 按照使用说明写好`docker-compose.yml`文件
3. `docker-compose up -d`

将我的`docker-compose.yml`共享在这里，参考docker hub 上的[embyserver](https://hub.docker.com/r/emby/embyserver)

```yaml
version: "3"
services:
  embyserver:
    image: emby/embyserver:latest
    ports:
      - 8096:8096
      - 8920:8920
    volumes:
      - /root/emby/config:/config
      - /root/emby/share1:/mnt/share1
      - /root/emby/share2:/mnt/share2
```

起来之后，通过浏览器访问[http://serverip:8096]()，经过一些简单的配置就可以使用起来了。

### 0x03 总结

其实，Emby可以用来作为家庭的媒体服务器，通过NAS搭建一个服务器，然后使用它提供的各类接入App，就能多终端接入了。

看到这篇文章的小伙伴如果有类似的更好的开源工具或解决方案，欢迎分享给我。
