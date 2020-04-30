---
title: 如何构建包含docker-client的镜像
date: 2018-04-12 16:52:02
categories: math
mathjax: true
tags: docker
---
**在很多场景下我们可能会在容器中调用docker命令,如何操作本文可以做一下简单介绍。**

- 选择基础镜像
可以安装docker client的系统很多,看你如何选择。为了精简镜像大小我选择了alpine作为基础镜像。
- 构建安装有docker客户端的镜像
1. Dockerfile
```
FROM alpine:3.5
RUN apk add --no-cache docker
```
2. 构建镜像
```
docker build -t registry.cn-hangzhou.aliyuncs.com/mojo/docker .

//查看一下构建出来的镜像大小
docker images | grep docker
输出:
registry.cn-hangzhou.aliyuncs.com/mojo/docker                    latest              cdf628f037a4        7 minutes ago       123MB

```
3. 验证
```
//启动容器
docker run -it --entrypoint /bin/sh -v /var/run/docker.sock:/var/run/docker.sock --name alpine registry.cn-hangzhou.aliyuncs.com/mojo/docker

//运行docker命令
docker ps
输出:
CONTAINER ID        IMAGE                                                                  COMMAND                  CREATED             STATUS              PORTS                    NAMES
bb80ea0ee311        registry.cn-hangzhou.aliyuncs.com/mojo/docker                          "/bin/sh"                3 seconds ago       Up 2 seconds                                 docker
6b3c5064ddf8        alpine                                                                 "/bin/sh"                23 minutes ago      Up 22 minutes                                alpine
8504459d3f00        dev_registry                                                           "/entrypoint.sh serve"   4 weeks ago         Up 12 hours         0.0.0.0:5000->5000/tcp   dev_registry_1
0b176b5e5983        registry.cn-hangzhou.aliyuncs.com/wise2c-hf/registry-database:v1.0.0   "docker-entrypoint.sh"   7 weeks ago         Up 12 hours         0.0.0.0:3306->3306/tcp   mysql
```