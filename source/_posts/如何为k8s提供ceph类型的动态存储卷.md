---
title: 如何为k8s提供ceph类型的动态存储卷
date: 2018-06-13 22:38:43
tags: k8s
---
本文针对在k8s中集成ceph并提供动态存储卷的流程做一个介绍。由于ceph安装比较麻烦,本文主要以docker形式安装ceph模块。动态存储卷的概念就不多介绍了,不了解的可以参考kubernetes[官网](https://v1-8.docs.kubernetes.io/docs/concepts/storage/persistent-volumes/)。使用的同学们需要注意kubernetes版本。kubernetes一直在更新中,不同版本的volume可能有区别,本文环境是基于kubernetes 1.8。

**注:**假设我们当前的k8s环境的node节点包含了(dev-8,dev-9,dev-10)

### ceph安装
#### docker形式启动ceph服务
```
docker run -d --net=host -v /etc/ceph:/etc/ceph -e MON_IP=192.168.1.189 -e CEPH_PUBLIC_NETWORK=192.168.1.0/24 ceph/demo ceph
```

**注:**确保每台主机都装有ceph client,如果没有装可以执行`yum install ceph -y`
#### 验证ceph是否正确安装
- 验证方式1
由于在启动容器的时候已经把容器里的目录挂载到了node节点上。所以直接在docker所在node上执行 rados lspools即可。

```
# rados lspools

rbd
cephfs_data
cephfs_metadata
.rgw.root
default.rgw.control
default.rgw.data.root
default.rgw.gc
default.rgw.lc
default.rgw.log
```

- 验证方式2
没有做volume的情况下可以进入容器执行上述命令

```
1. # docker ps | grep ceph

22216cb99087  ceph/demo  "/entrypoint.sh ceph" 27 minutes ago  Up 27 minutes stoic_hugle

2. # docker exec -it 22216cb99087 /bin/bash

3. # rados lspools

rbd
cephfs_data
cephfs_metadata
.rgw.root
default.rgw.control
default.rgw.data.root
default.rgw.gc
default.rgw.lc
default.rgw.log
```
#### 新建rbd pool
为了后面测试使用我们提前创建一个rbd pool(这里仅用于测试,较详细的ceph操作请移步ceph[文档](http://docs.ceph.com/docs/jewel/rados/operations/pools/)

```
ceph osd pool create kube 100 100
```

#### add ceph auth

```
ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'

// 查看刚刚add的ceph auth,并做base64处理
ceph auth get-key client.kube | base64

输出:
QVFCcjNpQmI5QmliSWhBQUtqcDRGYW5aem9YN1RXQXZFQndSbXc9PQ==
```

#### 拷贝ceph相关文件到k8s各node节点
由于k8s运行的pod可能存在于k8s任意节点,为了让k8s的各个节点都能正常与ceph通信,需要将ceph的相关目录拷贝的每个k8s node节点。(上文中提到了ceph的挂载目录,挂载目录包含了ceph的相关配置和可连接ceph的可执行文件)。假设当前所在节点为dev-10

```
scp -r /etc/ceph dev-9:/etc
scp -r /etc/ceph dev-8:/etc
```

拷贝完毕请登录到各个node执行 rados lspools 命令看当前node是否能正常连接到ceph。到目前为止ceph的相关准备工作已经完毕了。接下来继续k8s相关工作。

### k8s中的使用

下面主要介绍基于rbac模式的动态存储卷。如果用户觉得rbac模式麻烦,也可以采用把kubernetes admin.conf挂载到provisioner的相应目录,这样provisioner就有任意操作k8s集群的权限了。

- rbac模型定义
- secrets定义
- storageclass定义
- 测试pvc

#### rbac模型定义
1. 定义clusterrole.yaml

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rbd-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

```

2. 定义clusterrolebinding.yaml

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rbd-provisioner
subjects:
  - kind: ServiceAccount
    name: rbd-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: rbd-provisioner
  apiGroup: rbac.authorization.k8s.io

```

3. 定义serviceaccount.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-provisioner
```

4. 部署定义rbd类型的provisioner

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rbd-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: registry.cn-hangzhou.aliyuncs.com/mojo/rbd-provisioner
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccount: rbd-provisioner
```

**注:** 为了方便镜像下载registry.cn-hangzhou.aliyuncs.com/mojo/rbd-provisioner是我基于rbd ceph源码构建的docker镜像,并放于阿里云镜像仓库管理中心。

基于上面的步骤我大概做一个解释。上面定义了具备对整个系统的pv,pvc都有操作权限的clusterrole,并把clusterrole和指定的serviceaccount做了binding(就是上面的clusterrolebinding.yaml)。接着创建对应的serviceaccount.yaml,部署使用对应serviceaccount的rbd provisioner(即deployment.yaml)。这样,provisioner就具备对整个系统的pv,pvc操作权限了。

#### 测试
1. 定义secrets.yaml,secrets定义了一些具备操作ceph的ceph auth。provisioner会拿着这些认证去调用ceph相关api

```
apiVersion: v1
kind: Secret
metadata:
  name: ceph-admin-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth get-key client.admin | base64
  key: QVFBcDNpQmJ0RVpWRGhBQVdTMEovaVFUOHl4Q011dXlwRXg1OEE9PQ==
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
type: "kubernetes.io/rbd"
data:
  # ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'
  # ceph auth get-key client.kube | base64
  key: QVFCcjNpQmI5QmliSWhBQUtqcDRGYW5aem9YN1RXQXZFQndSbXc9PQ==
```

2. 定义storageclass.yaml, storageclass主要描述了ceph相关信息,让provisinor知道到那个地方使用什么auth去调用ceph api。

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 192.168.1.189:6789
  pool: kube
  adminId: admin
  adminSecretNamespace: kube-system
  adminSecretName: ceph-admin-secret
  userId: kube
  userSecretName: ceph-secret
  imageFormat: "2"
  imageFeatures: layering
```

#### 创建测试pvc

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rbd
  resources:
    requests:
      storage: 1Gi
```

查看pvc状态

```
kubectl get pvc

NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
rbd-claim1   Bound     pvc-b0327152-6ee9-11e8-afe2-00163e0f7d95   1Gi        RWO            rbd                   5h
```

到此为止整个流程已经操作完毕,如大家操作过程中遇到什么问题欢迎一起讨论。

#### 参考:
https://github.com/kubernetes-incubator/external-storage/tree/master/ceph/rbd