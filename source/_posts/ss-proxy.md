---
title: ss proxy
date: 2018-11-16 10:00:07
tags:
---

### docker-compose
```
version: '3'
services:
  ss-server:
    image: registry.cn-hangzhou.aliyuncs.com/shadow-go/ss-server:alpine
    ports:
    - 8388:8388
    volumes:
    - /root/docker-compose/proxy/config.json:/etc/config.json
```
### config.json
```
{
    "server":"server ip address",
    "server_port":8388,
    "local_port":1080,
    "password":"password",
    "method": "aes-128-cfb",
    "timeout":600,
    "fast_open":true
}
```