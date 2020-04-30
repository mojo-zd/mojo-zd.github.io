---
title: KT Connect一款基于k8s的开发利器
date: 2020-03-20 10:07:59
tags:
---
`KT Connect`作为一款开发环境治理的轻量级工具已经越来越受欢迎,官方文档非常详细请参考 https://alibaba.github.io/kt-connect/#/zh-cn 支持mac、linux、windows多种平台(windows由于诸多限制, 体验方面和其他平台相比略差)。经过最近一段时间的使用, 我想通过本篇文章对KT Connect的实现思路做一个梳理, 同时把这款好的工具安利给大家(有需求可以在githu上提issue,作者很勤快的)

## 前置条件
- [sshuttle](https://sshuttle.readthedocs.io/en/stable/overview.html)
- kubectl
- ktctl
- 本机具备访问k8s的能力

## 核心组件sshuttle | ssh
KT Connect基于sshuttle和ssh端口转发为开发者提供了强大的功能。

命令 | 功能 | 实现原理
---|---|---
connect | 本地访问集群服务(pod ip, dns都支持) | sshuttle(需要使用sudo权限)
exchange | 集群访问本地服务 | ssh远程端口转发
mesh | | ssh远程端口转发
run | 将本地服务暴露到集群中(集群中的其他服务可以通过podIP, 或者svc访问本地服务) | ssh远程端口转发

### ssh端口转发介绍
`ssh`端口转发也称为`ssh`隧道，因为他们通过`ssh`登录以后在`ssh`的客户端和服务器端建立一个隧道,从而进行通信。`ssh`的数据转发是经过加密的, 因此通过`ssh`隧道建立连接交换数据是非常安全的。`ssh`的端口转发可以分为多种, 下面介绍下常用的三种
#### 本地端口转发
将发送到本地端口的请求转发到目标主机端口, 从而达到访问目标主机端口的目的。

格式:

```ssh -L localPort:remoteHost:remotePort```

例:

```
// 本地主机登录到目标主机,通过访问本地2000端口访问目标主机的3000端口服务
ssh -L 2000:localhost:3000 root@xx.xx.xx.xx
```

#### 远程端口转发
将发送到远端主机端口的请求转发到目标主机,这样可以通过访问远端端口来达到访问目标主机端口的目的。

格式:

```ssh -R remotePort:targetHost:targetPort```

例:

```
// 本地登录到远程主机,通过访问远端2000端口访问本机的3000端口
ssh -R 2000:localhost:3000 root@xx.xx.xx.xx
```

#### 动态端口转发
针对本地端口转发和远程端口转发,都存在两个一一对应的端口，分别位于`ssh`的客户端和服务端,动态端口转发则只绑定一个本地端口,而targetHost:targetPort则是不固定的。targetHost:targetPort是由发起的请求决定的，比如，请求地址为`192.168.1.12:3000`，则通过`ssh`转发的请求地址也是`192.168.1.12:3000`。

格式:
```
ssh -D localhost:localPort
```
例:
```
// 本地主机登录到目标主机, 并进行动态端口转发
ssh -D localhost:3000 root@xx.xx.xx.xx
```
### KT Connect核心功能实现解析
KT Connect提供的几种服务都是通过在集群中部署`shadow`服务和本地的`ktctl`配合完成。其实可以理解`shadow`为`Server`,`ktctl`为`Client`,只要在这两个服务间建立起隧道即可解决本地与线上集群的网络问题。以下主机针对`connect、exchange、mesh、run`进行介绍。

### connect
`ktctl connect`利用`sshuttle`为开发者提供了本地访问集群服务的能力, 支持`ip`地址和`dns`两种。

执行`ktctl connect`, 过程如下:

1. 执行 `ktctl connect`, 部署`shadow`到k8s集群, 至于`shadow`部署到那个`namespace`使用什么镜像都可以自己指定。同时生成一个`ssh`的密钥对, 公钥挂载到`shadow`容器中。私钥保留在本地主机(后续使用sshuttle的时候需要用到私钥)
2. 执行`kubectl port-forward`命令, 使`shadow`服务的端口映射到主机端口
3. 执行`sshuttle`命令, 开启`dns`(`dns`地址即为`shadow`的`endpoint ip`)和`ssh`选项。建立起远端和本地的`ssh`隧道

到此`connect`的所有流程已经走完了。你可以在本地访问集群中任意服务的`ip:port`或者`dns:port`

## exchange
`ktctl exchange`将流量从本地引向集群, 也就是说可以在集群里面直接访问本地服务。有了这个功能直接省去了我们很多重复而且耗时的工作,比如打镜像,升级服务等等。使用如下:
- 运行一个本地服务 `docker run -itd -p 8080:80 nginx`
- 将本地服务暴露给集群`ktctl exchange tomcat --expose 8080`

过程如下:
1. 将已经存在的目标服务`tomcat`缩容为0
2. 重复`connect`步骤一,创建`shadow`服务并挂载公钥文件
3. 执行`kubectl port-forward`命令, 使`shadow`服务的端口映射到主机端口
4. 执行`ssh`的远程端口转发(`ssh -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -i privateKey -R  tomcatPort:127.0.0.1:localPort
        root@remoteHost -p remoteSSHPort`),上述例子中相当于把`tomcat`的流量转发到本地指定端口。

此时在集群里面原有的应用就被本地服务给替换了,从而达到`exchange`的目的

## mesh
`ktctl mesh`在`exchange`的基础上做了扩展, 并不是直接替换原有应用, 而是增加版本做规则路由。

执行`ktctl mesh tomcat --expose 8080`

过程如下:
1. 找出名为`tomcat`的`deploy`，基于该`deploy`标签进行复制,部署一个镜像为`shadow`的`deploy`。该`deploy`会以`tomcat-kt`加一个五位随机字符命名。而这个随机字符串就是该`deploy`的版本号
2. 执行kubectl `port-forward`命令, 使`shadow`服务的端口映射到主机端口
3. 执行ssh的远程端口转发，把远程指定端口流量指向本地应用
4. 此时万事具备, 只要修改`istio`路由规则就可以将满足条件的流量转向本地服务
## run
`ktctl run localservice --port 8080 --expose` 
其实原理同`exchange`一样, 唯一的区别在于没有替换集群中的服务。而是在集群中新建了一个`svc`, 用户可以通过`svc`或者`ip`访问到本地服务。


通过上述示例, 相信大家对`TK Connect`功能和细节有了一个大致了解。另外`KT Connect`还提供了可视化(dashboard)查看当前服务状态, check检查当前环境依赖, 更详细的使用请参考官网说明

### 总结
传统的自动化流水线已经很大程度提高了我们的研发效率, 但是对于开发环境来讲, 我们可能做一行代码的调整同样要走完整个自动化流水线,无疑浪费了宝贵的时间。`KT Connect`打破了网络限制实现了本地联调,真正解决了研发人员的痛点大大提高了研发效率,无疑为团队节约了大量的时间成本。