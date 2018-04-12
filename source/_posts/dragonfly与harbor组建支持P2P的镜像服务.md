---
title: dragonfly与harbor组建支持P2P的镜像服务
date: 2018-04-12 16:57:47
tags: registry
---
### 背景
Dragonfly是一个基于P2P的智能文件分发系统,用于解决大规模文件分发场景下分发耗时、成功率低、带宽浪费等难题。大幅提升发布部署、数据预热、大规模容器镜像分发等业务能力。为了提高容器镜像分发能力,我们选择了Dragonfly。本文主要基于Dragonfly来介绍一下配合Harbor如何搭建自己的支持P2P的镜像仓库服务(文件系统分发本文不做介绍)。
### 特性
- 基于P2P文件分发
- 支持各种容器化技术
- 主机级别限速策略
- 利用CDN机制避免远程重复下载
- 强一致性
- 磁盘保护,高效的IO处理
- 高性能
- 异常自动隔离
- 降低文件来源服务器压力
- 支持标准的Http Header
- 使用简单

### 架构介绍
- 整体流程

![image](http://ofp4ip1ms.bkt.clouddn.com/dfget-combine-container.png)


- block下载过程

![image](http://ofp4ip1ms.bkt.clouddn.com/distributing.png)

**流程解析:**
1. 当执行docker pull操作时,dfget-proxy会拦截docker pull请求。将请求转发给CM(cluster manager)。
> cm的地址已经在client主机的/etc/dragonfly.conf文件中配置好了。另外上文中提到的dfget-proxy其实就是df-daemon。Dragonfly中有三个项目,client端:getter(python)、daemon(golang),docker pull时,df-daemon拦截到请求并通过dfget进行文件拉取,server端:supernode(java)。

2. df-daemon启动的时候带了registry参数,并且通过dfget传给服务端supernode。supernode解析参数到对应的镜像仓库获取镜像并以block的形式返回给客户端。如果再次拉取镜像时,supernode就会检测哪一个client存在和镜像文件对应的block,如果存在直接从该client下载,如果不存在就通过server端到镜像仓库拉取镜像。

### 安装
#### install server
Dragonfly官方支持基于Docker和Physical Machine两种方案部署server。为了快速搭建,本文基于Docker的方式来安装server。也可参考[官方文档](https://github.com/alibaba/Dragonfly/blob/master/docs/install_server.md)进行安装


1. clone
```
git clone https://github.com/alibaba/Dragonfly.git
```

2. build image
```
cd dragonfly
docker build -t "dragonfly:supernode" . -f ./build/supernode/Dockerfile
```

3. run dragonfly container
```
docker run -d -p 8001:8001 -p 8002:8002 dragonfly:supernode
```
> 由于基于源码构建的镜像比较大,比较耗时。为了使用方便,我基于源码构建了一个公有的镜像置于阿里云镜像仓库服务器.不想自己构建镜像的小伙伴可以直接使用如下方式启动server。
```
docker run -d -p 8001:8001 -p 8002:8002 --restart=always registry.cn-hangzhou.aliyuncs.com/p2p-service/dragonfly-server
```
#### install client
1. 下载
```
wget https://github.com/alibaba/Dragonfly/raw/master/package/df-client.linux-amd64.tar.gz
tar -zxvf df-client.linux-amd64.tar.gz -C install
```
> install为解压的目录,稍后会将这个目录添加到环境变量。
2. 设置环境变量path
```
echo $SHELL (get which shell using, current shell is bash)
vi ~/.bashrc

将下面的设置添加到~/.bashrc文件末
PATH=$PATH:xxx/xxx/install/df-client

退出vi并执行以下命令
source ~/.bashrc
```
3. 验证df-client安装是否正确
```
dfget -h

usage: dfget [-h] [--url URL] [--output OUTPUT] [--md5 MD5]
             [--callsystem CALLSYSTEM] [--notbs] [--locallimit LOCALLIMIT]
             [--totallimit TOTALLIMIT] [--identifier IDENTIFIER]
             [--timeout TIMEOUT] [--filter FILTER] [--showbar]
             [--pattern {p2p,cdn}] [--version] [--node NODE] [--console]
             [--header HEADER] [--dfdaemon]

dragonfly is a file distribution system based p2p

optional arguments:
  -h, --help            show this help message and exit
  --url URL, -u URL     will download a file from this url
  --output OUTPUT, -O OUTPUT, -o OUTPUT
```
### Harbor搭建
> 基于在线方式的安装, 本文采用http的方式配置Harbor。Harbor版本为1.2.2
1. 下载 
```
wget https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-online-installer-v1.2.2.tgz

tar -zxvf harbor-online-installer-v1.2.2.tgz -C install
```

2. 修改配置
```
cd install/harbor
vi harbor.cfg
hostname=xx.xx.xx.xx  //设置为当前主机ip
```
3. 安装并启动harbor服务
> 服务启动以后,如果需要管理Harbor服务的生命周期,可以直接通过docker-compose来管理
```
sh install.sh
```
**说明:**
执行以上脚本做了一下几件事情
- 根据harbor.cfg配置和template模板生成对应的配置文件到common目录下
- 调用docker-compose up -d 启动harbor相关服务。第一执行会自动下载相关镜像

4. 直接访问 http://xx.xx.xx.xx 默认账号为 admin:Harbor12345

### 使用指南
1. 在client主机上通过配置文件指定CM(cluster manager)节点
```
vi /etc/dragonfly.conf
内容:
[node]
address=cm node host ip
```
2. 由于当前Dragonfly暂不支持harbor认证。如果按照[官网](https://github.com/alibaba/Dragonfly/blob/master/docs/usage.md)配置"configure daemon mirror"来拉取镜像会提示授权失败。为了绕过这个问题可以采用docker proxy的方式来解决。具体步骤如下:
- vi /etc/systemd/system/docker.service.d/http-proxy.conf,没有该文件就直接创建该文件。通过添加proxy，在拉取镜像时将会通过下面的配置地址转发到目标机。
```
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:65001"
```
- 更新变更
```
systemctl daemon-reload
```
- 验证配置是否生效
```
systemctl show --property=Environment docker
输出:
Environment=GOTRACEBACK=crash DOCKER_HTTP_HOST_COMPAT=1 PATH=/usr/libexec/docker:/usr/bin:/usr/sbin HTTP_PROXY=http://127.0.0.1:65001
```
3. 在client机上添加harbor的insecure地址
- vi /etc/docker/daemon.json
![image](http://ofp4ip1ms.bkt.clouddn.com/QQ20180312-161402.png)把harbor地址添加到标红模块处
- 重启docker
```
systemctl restart docker
```
4. 验证inscure是否生效
```
docker login --username=admin xx.xx.xx.xx
```
登录成功说明inscure设置生效。

5. 启动client服务
```
df-daemon --registry http://xx.xx.xx.xx
```
6. 验证
```
1. tcpdump -i lo port 65001
2. docker pull xx.xx.xx.xx/repo/image:tag //拉取镜像
```
效果(有数据经过65001端口):
```
09:56:54.380443 IP localhost.56894 > localhost.65001: Flags [P.], seq 1:1762, ack 1, win 342, options [nop,nop,TS val 3956769309 ecr 3956769308], length 1761
09:56:54.380466 IP localhost.65001 > localhost.56894: Flags [.], ack 1762, win 1365, options [nop,nop,TS val 3956769309 ecr 3956769309], length 0
09:56:54.412983 IP localhost.65001 > localhost.56894: Flags [P.], seq 1:4097, ack 1762, win 1365, options [nop,nop,TS val 3956769341 ecr 3956769309], length 4096
09:56:54.413001 IP localhost.56894 > localhost.65001: Flags [.], ack 4097, win 1365, options [nop,nop,TS val 3956769341 ecr 3956769341], length 0
09:56:54.413022 IP localhost.65001 > localhost.56894: Flags [P.], seq 4097:4352, ack 1762, win 1365, options [nop,nop,TS val 3956769341 ecr 3956769341], length 255
09:56:54.413029 IP localhost.56894 > localhost.65001: Flags [.], ack 4352, win 1429, options [nop,nop,TS val 3956769341 ecr 3956769341], length 0
09:56:54.413049 IP localhost.65001 > localhost.56894: Flags [F.], seq 4352, ack 1762, win 1365, options [nop,nop,TS val 3956769341 ecr 3956769341], length 0
09:56:54.413192 IP localhost.56894 > localhost.65001: Flags [F.], seq 1762, ack 4353, win 1429, options [nop,nop,TS val 3956769341 ecr 3956769341], length 0
09:56:54.413212 IP localhost.65001 > localhost.56894: Flags [.], ack 1763, win 1365, options [nop,nop,TS val 3956769341 ecr 3956769341], length 0
09:56:54.413791 IP localhost.56896 > localhost.65001: Flags [S], seq 2279785798, win 43690, options [mss 65495,sackOK,TS val 3956769342 ecr 0,nop,wscale 7], length 0
16:35:45.400349 IP localhost.65001 > localhost.56896: Flags [S.], seq 127072263, ack 2279785799, win 43690, options [mss 65495,sackOK,TS val 3956769342 ecr 3956769342,nop,wscale 7], length 0
09:56:54.413819 IP localhost.56896 > localhost.65001: Flags [.], ack 1, win 342, options [nop,nop,TS val 3956769342 ecr 3956769342], length 0
09:56:54.413984 IP localhost.56896 > localhost.65001: Flags [P.], seq 1:374, ack 1, win 342, options [nop,nop,TS val 3956769342 ecr 3956769342], length 373
09:56:54.413995 IP localhost.65001 > localhost.56896: Flags [.], ack 374, win 350, options [nop,nop,TS val 3956769342 ecr 3956769342], length 0
09:56:54.444136 IP localhost.65001 > localhost.56896: Flags [P.], seq 1:1230, ack 374, win 350, options [nop,nop,TS val 3956769372 ecr 3956769342], length 1229
09:56:54.444153 IP localhost.56896 > localhost.65001: Flags [.], ack 1230, win 1365, options [nop,nop,TS val 3956769372 ecr 3956769372], length 0
09:56:54.444180 IP localhost.65001 > localhost.56896: Flags [F.], seq 1230, ack 374, win 350, options [nop,nop,TS val 3956769372 ecr 3956769372], length 0
09:56:54.444360 IP localhost.56896 > localhost.65001: Flags [F.], seq 374, ack 1231, win 1365, options [nop,nop,TS val 3956769373 ecr 3956769372], length 0
09:56:54.444379 IP localhost.65001 > localhost.56896: Flags [.], ack 375, win 350, options [nop,nop,TS val 3956769373 ecr 3956769373], length 0
09:56:54.444695 IP localhost.56898 > localhost.65001: Flags [S], seq 3831704766, win 43690, options [mss 65495,sackOK,TS val 3956769373 ecr 0,nop,wscale 7], length 0
16:37:58.544366 IP localhost.65001 > localhost.56898: Flags [S.], seq 3015556728, ack 3831704767, win 43690, options [mss 65495,sackOK,TS val 3956769373 ecr 3956769373,nop,wscale 7], length 0
09:56:54.444718 IP localhost.56898 > localhost.65001: Flags [.], ack 1, win 342, options [nop,nop,TS val 3956769373 ecr 3956769373], length 0
09:56:54.444835 IP localhost.56898 > localhost.65001: Flags [P.], seq 1:1692, ack 1, win 342, options [nop,nop,TS val 3956769373 ecr 3956769373], length 1691
09:56:54.444849 IP localhost.65001 > localhost.56898: Flags [.], ack 1692, win 1365, options [nop,nop,TS val 3956769373 ecr 3956769373], length 0
09:56:54.477140 IP localhost.65001 > localhost.56898: Flags [P.], seq 1:4097, ack 1692, win 1365, options [nop,nop,TS val 3956769405 ecr 3956769373], length 4096
09:56:54.477158 IP localhost.56898 > localhost.65001: Flags [.], ack 4097, win 1365, options [nop,nop,TS val 3956769405 ecr 3956769405], length 0
09:56:54.477172 IP localhost.65001 > localhost.56898: Flags [P.], seq 4097:4352, ack 1692, win 1365, options [nop,nop,TS val 3956769405 ecr 3956769405], length 255
```
> 如果要看日志可以在根目录.small-dragonfly/logs下查看
