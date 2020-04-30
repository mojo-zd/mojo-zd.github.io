---
title: 如何在项目中集成kubectl
date: 2018-05-14 15:54:04
tags: k8s
---

#### 为什么需要在项目中集成kubectl工具
开发模块中涉及到对k8s的资源操作,操作方式除了api调用以外也可以调用apply yaml的形式。
由于k8s版本更新比较频繁,因此api方式调用k8s资源也会经常调整。为了简单化管理可以采用调用kubectl的形式进行与k8s交互。那么下面就给大家介绍一下如何集成kubectl(本文是基于容器介绍)
#### 如何集成
1. 新建template模板文件(kubeconf.gotmpl),kubectl需要权限才能操作k8s相关资源,下面的内容是基于k8s的/etc/kubernetes/admin.conf而来
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: {{.CAData}}
    server: {{.Endpoint}}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: {{.CertData}}
    client-key-data: {{.KeyData}}
```

2. 根据模板生成对应的文件
```
import (
    "bytes"
    "html/template"
    "io/ioutil"
    "os"
    "os/exec"
    "strings"

    "wise-resource-manager/pkg/domain"

    "github.com/astaxie/beego/logs"
)

var (
    DIR_TEMP             = "/tmp"
    FILE_PREFIX_KUBECONF = "wisecloud-kubeconf-"
)

func init() {
    os.MkdirAll(DIR_TEMP, 0777)
}
// 
func Run(args string, environment *domain.SampleEnvironment) (string, error) {
    logs.Debug("args: ", args)

    content, err := parseConfFile("/kubeconf.gotmpl", environment)
    if err != nil {
        logs.Error(err.Error())
        return "", err
    }

    path, err := newConfFile(content)
    if err != nil {
        logs.Error(err.Error())
        return "", err
    }
    defer os.RemoveAll(path)

    out, err := execute(args, path)
    if err != nil {
        logs.Error(err.Error())
        return "", err
    }
    return out, nil
}

// 指定kubeconfig环境变量并执行kubectl命令
func execute(args string, path string) (string, error) {
    var (
        out    bytes.Buffer
        stderr bytes.Buffer
    )

    if err := os.Setenv("KUBECONFIG", path); err != nil {
        logs.Error(err.Error())
        return out.String(), err
    }

    argList := strings.Split(args, " ")
    cmd := exec.Command(argList[0], argList[1:]...)
    cmd.Stdout = &out
    cmd.Stderr = &stderr
    err := cmd.Run()
    if err != nil {
        logs.Error(err.Error(), stderr.String())
        return stderr.String(), err
    }
    return out.String(), nil
}

// 根据模板生成文件
func parseConfFile(tpl string, obj interface{}) (string, error) {
    var buf bytes.Buffer

    t, err := template.ParseFiles(tpl)
    if err != nil {
        logs.Error(err.Error())
        return string(buf.Bytes()), err
    }
    if err := t.Execute(&buf, obj); err != nil {
        return string(buf.Bytes()), err
    }

    return string(buf.Bytes()), nil
}

// 生成文件到指定目录
func newConfFile(content string) (path string, err error) {
    file, err := ioutil.TempFile(DIR_TEMP, FILE_PREFIX_KUBECONF)
    if err != nil {
        logs.Error("create kubeconf temp file error!")
        return "", err
    }
    defer file.Close()

    path = file.Name()
    file.WriteString(content)
    return
}
```
3. 打包包含kubectl的镜像
```
FROM centos:7
MAINTAINER excessivespeed@126.com

RUN curl -SL http://p1c71d06h.bkt.clouddn.com/kubectl-v1.8.6 -o /usr/bin/kubectl-v1.8.6 && \
    curl -SL http://p1c71d06h.bkt.clouddn.com/kubectl-v1.7.3 -o /usr/bin/kubectl-v1.7.3 && \
    chmod 777 /usr/bin/kubectl-v1.8.6 && \
    chmod 777 /usr/bin/kubectl-v1.7.3 && \

COPY ./kubeconf.gotmpl /
COPY ./wise-resource-manager /go/bin/wise-resource-manager

RUN chmod u+x /go/bin/wise-resource-manager

WORKDIR /go/bin/

RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENTRYPOINT ["/go/bin/wise-resource-manager"]
```

#### 验证方式
1. 用户可以通过运行镜像提供简单的api调用方式验证。
2. 保证gotmpl文件能正确生成的情况下,直接在容器任意目录新建一个任意文件.这里以wisecloud-kubeconf.conf为例。
```
docker exec -it docker-container /bin/sh //进入容器
pwd  //假设当前目录为/bin/go
vi wisecloud-kubeconf.conf //拷贝/etc/kubernetes/admin.conf文件内容到这里
export KUBECONFIG=/bin/go/wisecloud-kubeconf.conf
kubectl get nodes //列出当前集群的所有节点
```