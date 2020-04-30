---
title: frp内网穿透神器
date: 2018-04-21 10:48:06
tags: network

---
#### 为什么使用内网穿透
很多场景在调用第三方API的时候需要填写回调地址,如果在内网开发的话就需要做内网穿透。今天给大家介绍一款内网穿透工具[frp](https://github.com/fatedier/frp),使用起来既简单又方便。

#### 环境

name | Version
---|---
centos | 7.3

#### 安装步骤
1. [下载](https://github.com/fatedier/frp/releases)压缩包
2. 解压
```
tar -zxvf frp_0.14.0_linux_amd64.tar.gz
```
3. 配置相关参数实现通过ssh访问公司内网机器
- server端配置
```
vi frps.ini

[common]
bind_port = 7000
vhost_http_port = 8079
```
- client端配置 (xx.xx.xx.xx为外网可访问的IP)
```
vi frpc.ini

[common]
server_addr = xx.xx.xx.xx 
server_port = 7000

[ssh]
local_port = 22
remote_port = 6000

[web]
type = http
local_port = 8079
custom_domains = xx.xx.xx.xx
```
4. 启动相关服务
- start server
```
./frps -c ./frps.ini
```

- start client
```
./frpc -c ./frpc.ini
```
5. 验证(通过ssh访问内网机器,假设用户名为test)
```
ssh -oPort=6000 test@xx.xx.xx.xx
```
