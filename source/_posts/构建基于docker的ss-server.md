---
title: 构建基于docker的ss server
date: 2018-05-28 23:34:01
tags: docker
---

#### 为什么需要构建docker版的ss server
这几年容器技术发展的相当迅猛,个人也是一个容器爱好者。只要一次构建镜像一个命令就可以拉起来一个ss server, 所以构建了基于容器的ss server。下面构建基于源代码地址为 https://github.com/mojo-zd/shadowsocks-go

#### 构建并运行server
1. 进入server目录
```
cd shadowsocks-go/cmd/shadowsocks-server
```
2. 
```
make build //构建基于linux的可执行文件
make docker-build //构建docker镜像,并推送到阿里云镜像仓库
```
3. 运行docker-compose启动server
```
//cmd/shadowsocks-server下有对应的docker-compose文件和需要的config.json文件只需要修改成实际环境参数即可
docker-compose up -d
```
注: ss server镜像我已经推送的阿里云的公开镜像仓库了。各位客官可以直接配置config.json并运行docker-compose.yaml