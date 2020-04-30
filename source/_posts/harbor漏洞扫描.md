---
title: harbor漏洞扫描
date: 2018-05-07 15:44:33
tags: registry
---

### Harbor漏洞扫描实现
Harbor通过开源组件Clair实现了漏洞扫描功能,Clair提供了静态文件扫描功能。harbor中可以对特定镜像或者所有镜像进行漏洞扫描，此外用户还可以通过设定扫描策略实现针对所有镜像的漏洞扫描。

Clair依赖于vulnerability metadata来实现分析过程.第一次初始化安装以后,Clair会自动更新metadata。更新时间取决于data大小和network连接情况.如果数据库并未完全准备好,harbor中的镜像列表页面footer位置将会有警告信息.

### 基于clair启动harbor
harbor提供了基于clair的启动模式,启动非常简单
```
1. cd $HARBOR_ROOT //进入harbor根目录
2. ./install.sh --with-clair
```