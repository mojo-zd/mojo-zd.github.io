---
title: install nfs server
date: 2018-04-12 17:07:38
tags: storage driver
---
### nfs服务器安装
##### ENV: Linux - CentOS-7
##### Setup 1: Install package
```
yum install nfs-utils -y
```
##### Setup 2: Create DIR & Configuration
```
mkdir /var/nfsshare
echo "/var/nfsshare 182.140.210.0/24(rw,sync,no_root_squash)" >> /etc/exports
multi host eg: /var/nfsshare 121.40.169.81/32(rw,sync,no_root_squash) 101.37.26.244/32(rw,sync,no_root_squash) 121.40.232.51/32(rw,sync,no_root_squash) 101.37.38.115/32(rw,sync,no_root_squash)
```
##### Setup 3: Start Service
```
systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap 
 
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap
```
##### Setup 4: Check
```
mount -t nfs 182.140.210.216:/var/nfsshare /mnt  -o vers=3,nfsvers=3
mkdir -p /mnt/xx
ls /var/nfsshare
umount /mnt
```