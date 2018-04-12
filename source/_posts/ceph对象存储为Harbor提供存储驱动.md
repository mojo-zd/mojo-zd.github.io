---
title: ceph对象存储为Harbor提供存储驱动
date: 2018-04-12 17:04:27
tags: registry
---
### 主机环境

name | Version
---|---
centos | 7.3
docker server | 1.12.6
docker client | 1.12.6
docker-compose | 1.19.0

### ceph服务器
#### 启动服务
```
docker run -d \
                -e MON_IP=127.0.0.1 \
                -e CEPH_NETWORK=127.0.0.0/24 \
                -e RGW_CIVETWEB_PORT=8080 \
                --net=host \
                --name=ceph-demo \
                kiwenlau/ceph-demo
```
> 如果RGW_CIVETWEB_PORT不指定,默认监听7480端口

#### 添加swift用户
- 创建s3用户
```
radosgw-admin user create --uid="mojo" --display-name="Mojo" --email=test@test.com
```
输出
```
{
    "user_id": "mojo",
    "display_name": "Mojo",
    "email": "test@test.com",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "mojo",
            "access_key": "3AFLMXSA3WUT5NVDC8Z6",
            "secret_key": "3VY3SrcLjDA8ReiswdOvqODXUawyshZ3txTNFF5A"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "max_size_kb": -1,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "max_size_kb": -1,
        "max_objects": -1
    },
    "temp_url_keys": []
}
```
- 添加子账号(swift api使用)
```
radosgw-admin subuser create --uid=mojo --subuser=mojo:swift --access=full
```
输出:
```
{
    "user_id": "mojo",
    "display_name": "Mojo",
    "email": "test@test.com",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [
        {
            "id": "mojo:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "mojo",
            "access_key": "3AFLMXSA3WUT5NVDC8Z6",
            "secret_key": "3VY3SrcLjDA8ReiswdOvqODXUawyshZ3txTNFF5A"
        }
    ],
    "swift_keys": [
        {
            "user": "mojo:swift",
            "secret_key": "LRDTpRiMXjC1fKWBQwPrP327myVnZ2FSB0XGAkV8"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "max_size_kb": -1,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "max_size_kb": -1,
        "max_objects": -1
    },
    "temp_url_keys": []
}
```
> 添加swift账号必须先创建s3用户

### client主机操作
#### 安装swift客户端
```
1. yum install python-setuptools
2. easy_install pip
3. pip install --upgrade setuptools
4. pip install --upgrade python-swiftclient
```

#### 创建registry container
```
swift -A http://192.168.1.251:8080/auth/v1.0 -U mojo:swift -K 'LRDTpRiMXjC1fKWBQwPrP327myVnZ2FSB0XGAkV8' post registry
```
> 192.168.1.251 为ceph服务所在主机ip, 8080为RGW_CIVETWEB_PORT的值

#### 查看创建的registry container
```
swift -A http://192.168.1.251:8080/auth/v1.0 -U mojo:swift -K 'LRDTpRiMXjC1fKWBQwPrP327myVnZ2FSB0XGAkV8' list
```
输出:
```
registry
```

### Harbor安装
> 基于在线方式的安装, 本文采用http的方式配置harbor
- 下载
```
1. wget https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-online-installer-v1.2.2.tgz
2. tar -zxvf harbor-online-installer-v1.2.2.tgz -C install
```
- 修改配置
```
1. install/harbor
2. vi harbor.cfg
hostname=xx.xx.xx.xx  //设置为当前主机ip
```
- 修改harbor registry配置文件
```
vi common/templates/registry/config.yml
```
内容:
```
version: 0.1
log:
  level: debug
  fields:
    service: registry
storage:
  swift:
    username: mojo:swift
    password: LRDTpRiMXjC1fKWBQwPrP327myVnZ2FSB0XGAkV8
    authurl: http://192.168.1.251:8080/auth/v1.0
    tenantid: mojo
    container: registry
    insecureskipverify: false
    rootdirectory: /registry
http:
    addr: :5000
    secret: placeholder
    debug:
        addr: localhost:5001
auth:
  token:
    issuer: harbor-token-issuer
    realm: $ui_url/service/token
    rootcertbundle: /etc/registry/root.crt
    service: harbor-registry
notifications:
  endpoints:
      - name: harbor
        disabled: false
        url: http://ui/service/notifications
        timeout: 3000ms
        threshold: 5
        backoff: 1s
```
> authurl为http://CEPH_SERVER_IP:RGW_CIVETWEB_PORT/auth/v1.0
- 为docker添加insecure
```
1. vi /etc/docker/daemon.json

{
  "registry-mirrors": ["http://ef017c13.m.daocloud.io"],
  "exec-opt": [
    "native.cgroupdriver=systemd"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "insecure-registries": [
      "xx.xx.xx.xx" //编辑daemon.json文件仅需要修改此处,此处为添加的harbor的ip地址
  ],
  "storage-driver": "overlay"
}

2. systemctl daemon-reload
3. systemctl restart docker
```
- 下载harbor镜像并启动harbor服务
```
sh install.sh
```

### 验证
- 进入ceph容器list pool
```
1. docker exec -it ceph-demo /bin/bash
2. rados lspools
```
输出:
```
rbd
cephfs_data
cephfs_metadata
.rgw.root
.rgw.control
.rgw
.rgw.gc
.log
.users.uid
.users.email
.users
.users.swift
.rgw.buckets.index
```
- 在client端push新的镜像到镜像仓库
```
1. docker login --username=admin xx.xx.xx.xx
2. docker tag docker.io/nginx xx.xx.xx.xx/library/nginx
3. docker push xx.xx.xx.xx/library/nginx
```
> xx.xx.xx.xx为harbor的访问地址

- 验证镜像是否已经存在ceph中
```
1. docker exec -it ceph-demo /bin/bash
2. rados lspools
```
输出:
```
rbd
cephfs_data
cephfs_metadata
.rgw.root
.rgw.control
.rgw
.rgw.gc
.log
.users.uid
.users.email
.users
.users.swift
.rgw.buckets.index
.rgw.buckets
```
和第一次rados lspools对比发现现在多了一个.rgw.buckets pool。查看.rgw.buckets pool中是否存在刚刚推送的镜像文件
```
rados --pool .rgw.buckets ls | grep library/nginx
```
输出:
```
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_manifests/revisions/sha256/600bff7fb36d7992512f8c07abd50aac08db8f17c94e3c83e47d53435a1a6f7c/link
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_layers/sha256/4e9f6296fa34712fd20dff7940a476b0d47b91671528644253336bfb00fb4924/link
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_manifests/tags/latest/current/link
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_layers/sha256/5b19c1bdd74be0e4a37550f5b05fbc2280961123634f58da0765bbc36b71f444/link
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_manifests/tags/latest/index/sha256/600bff7fb36d7992512f8c07abd50aac08db8f17c94e3c83e47d53435a1a6f7c/link
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_layers/sha256/e548f1a579cf529320a3c1d8789399e5fea0bfaa04cbb70d03890afafb748a2f/link
default.4108.1_files/docker/registry/v2/repositories/library/nginx/_layers/sha256/8176e34d5d92775e15a602541e02fec25a22933a12561c114436b757b8e7a9e8/link
```

### 对比测试
#### 第一组测试
- 条件

image name | size
---|---
library/centos | 9.238G

- loacl storage

command | start at | end at | spend
---|---|---|---
docker push 139.159.228.163/library/centos | 11:24:48 | 11:28:26 | 3m 38s
docker pull 139.159.228.163/library/centos | 14:09:24 | 14:12:06 | 2m 42s

- ceph swift

command | start at | end at | spend
---|---|---|---
docker push 139.159.230.56/library/centos | 11:16:09 | 11:20:28 | 4m 19s
docker pull 139.159.230.56/library/centos | 14:17:28 | 14:21:43 | 4m 15s

#### 第二组测试
- 条件

image name | size
---|---
library/fedora | 2.201G

- loacl storage

command | start at | end at | spend
---|---|---|---
docker push 139.159.228.163/library/fedora | 14:36:12 | 14:36:50 | 38s
docker pull 139.159.228.163/library/fedora | 14:41:31 | 14:42:17 | 46s

- ceph swift

command | start at | end at | spend
---|---|---|---
docker push 139.159.230.56/library/fedora | 14:39:03 | 14:40:01 | 58s
docker pull 139.159.230.56/library/fedora | 14:43:56 | 14:45:03 | 1m 07s