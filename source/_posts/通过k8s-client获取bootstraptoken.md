---
title: 通过k8s client获取bootstraptoken
date: 2018-05-25 10:59:38
tags: k8s
---

#### bootstraptoken 用来做什么
安装kubeadm以后,执行kubeadm init,控制台输出会提示:
```
[bootstraptoken] Using token: <token>
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
```
k8s中join节点的时候通常需要用到这个token。那么如何获取到这个值呢

#### bootstraptoken 是怎么存储的
bootstraptoken 是以secrets的形式存储在kube-system namespace下。secretsdata结构并不固定,经过查看源码发现,bootstraptoken token在secrets中存储时有一下特殊字段。

```
token-id、token-secret
```
#### 获取token
```
func BootstrapToken(environment *model.Environment) string {
    secrets, err := ListSecret(environment, metaV1.NamespaceSystem)
    var tokenId, tokenSecret string
    if err != nil {
        return ""
    }
    for _, secret := range secrets.Items {
        tokenId = getSecretString(&secret, BootstrapTokenIDKey)
        if len(tokenId) == 0 {
            continue
        }

        tokenSecret = getSecretString(&secret, BootstrapTokenSecretKey)
        if len(tokenSecret) > 0 {
            break
        }
    }

    return helper.ParseKubeToken(tokenId, tokenSecret)
}


func ParseKubeToken(tokenId, secret string) string {
    return fmt.Sprintf("%s.%s", tokenId, secret)
}

func getSecretString(secret *v1.Secret, key string) string {
    if secret.Data == nil {
        return ""
    }
    if val, ok := secret.Data[key]; ok {
        return string(val)
    }
    return ""
}
```