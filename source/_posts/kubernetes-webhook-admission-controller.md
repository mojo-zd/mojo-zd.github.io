---
title: kubernetes webhook admission controller
date: 2019-06-28 10:47:56
tags: k8s
---
### What is admission webhooks
kubernetes的admission webhooks为开发者提供了非常灵活的插件模式,在kubernetes资源持久化之前用户可以对指定资源做校验、修改等操作。应用场景 eg：接入sidecare、设置默认的servceaccount、设置quota等等

### How it works
- 注册webhook server
- 资源操作请求通过API Server Auth验证
- 根据注册信息回调对应的webhook server

**webhook注册信息说明**
```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: config
webhooks:
- name: webhook.default.svc ①
  rules: ②
  - apiGroups:
    - "*"
    apiVersions:
    - "*"
    operations:
    - CREATE
    resources:
    - deployments 
  clientConfig:
    service:
      namespace: default
      name: webhook
      path: /pods ③
    ④caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUREekNDQWZlZ0F3SUJBZ0lKQU0va1pKalJKaFpuTUEwR0NTcUdTSWIzRFFFQkN3VUFNQjR4SERBYUJnTlYKQkFNTUUzZGxZbWh2YjJzdVpHVm1ZWFZzZEM1emRtTXdIaGNOTVRrd05qSTNNRGd4TkRBeldoY05Namt3TmpJMApNRGd4TkRBeldqQWVNUnd3R2dZRFZRUUREQk4zWldKb2IyOXJMbVJsWm1GMWJIUXVjM1pqTUlJQklqQU5CZ2txCmhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBcUdvVG05eGNMNmhTMTBGVGNlc0FnTGszNGh2NXEyWUUKb0YxdDZ6ZlVFd2tRZ3BHem9FNmJDSjVIb0luTnhQWHAxRlN4RVpVQ2crWk8xWCtIY0tReHRQdmpWT1JqYkUzcQpOVFh5NmgyNEUzVXQxNDR0UVM2MHVuakVpNmxMZjFsYmNyNXJ4MG5UVFNNKzZNWWF3SWRNR1VzU29QWTIzUDRiCmsxYlQ2N09mZGF3c1A0UFFMZG9nY3p2aDJ2SGl2UEtQNllyY3ZMNHFUN2twYzllYWdodGVIVlA0ZXRHZFp6c1gKUXhwNUgwdVFQaml0TE9sTkwwT25iRURFWkRIaDRUVGpkc2d6TXVBbXRFNVhtN1lCbWlNRFRIZkFLVUVIdEpUZwo0b1A2YnVTYyswZFJvY0gyMG5mQzI5Y1RXNDdMc3RSbVV4MEhNRGVnTG04a1JEeTZJRnkyRFFJREFRQUJvMUF3ClRqQWRCZ05WSFE0RUZnUVVrdGxEdnc1TUF1Z0ZtVkN6MGUweEdXRHNsZnd3SHdZRFZSMGpCQmd3Rm9BVWt0bEQKdnc1TUF1Z0ZtVkN6MGUweEdXRHNsZnd3REFZRFZSMFRCQVV3QXdFQi96QU5CZ2txaGtpRzl3MEJBUXNGQUFPQwpBUUVBQk1pbVNieW9IeGlIVkYyOGczU3ZrNDdBNkUvNEttcEdoTlFIMmh1QStmczhObFc0MXlrbmQwbFMyT0RJCnU1aks2MEdMaTdQTmszeFc4bXBERnJ5OTh3d1BIV1dRUVh0WEVEa3dnaFl2Y1ErVy9JbFpJQXdFcTAyOE5kLzUKVnc4ZWNWaFhFNkNHdmszNmVMQ3ZSeDQvOVRGeDg3Z0hpdnl2V3YxcGJQMjFqWjQ1MVBsZDUyYkpRek1zNHg3QworekN5d3k4NG9QTjBsYzduMC9YdlJTWi9hbVVEQzhtaGd3YVFsUGpxd0FJUU9GL1F5QkdFQ2N0SjNHZU91cGhpCnJPUzBrZGs4YzFyLzh6bVp1Q1hNWTZKc21saGhuMCtpWHhDdXRhRWdpeWxxdEZTM0NLZkdmSnN3b2M5QjN5WHUKQWlHc21uWVoxbEoyRkdmWU1lT2xoZzJlK1E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0t
```
① webhook名称

② 描述api-server操作什么资源什么动作时调用webhook插件

③ 调用webhook api的地址

④ 提供和webhook通信的TLS链接信息

注: 如何编写webhook server可以参考 https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go
### Require
- 准备一个kubernetes集群必须为v1.9或以上的版本
- api server需要开启`MutatingAdmissionWebhook` `ValidatingAdmissionWebhook`，通过一下命令可查看
```
kubectl api-versions | grep admissionregistration
> admissionregistration.k8s.io/v1beta1
```

上面提到了webhook需要使用TLS, 为了简单我们可以使用cert-manager来管理证书生成
### Install cert-manager
cert-manager的安装非常简单,官网也给出了步骤。
- Installing with regular manifests
``` 
# Create a namespace to run cert-manager in
kubectl create namespace cert-manager

# Disable resource validation on the cert-manager namespace
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Install the CustomResourceDefinitions and cert-manager itself
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.8.1/cert-manager.yaml
```
注: 安装过程中请注意文档中的Note,我当前执行的环境为k8s v1.13.6,本文给出的步骤只支持kubernetes v1.12以上。如果遇到以下问题请再执行一遍`apply`
```
unable to recognize "https://github.com/jetstack/cert-manager/releases/download/v0.8.1/cert-manager.yaml": no matches for kind "Issuer" in version "certmanager.k8s.io/v1alpha1"
unable to recognize "https://github.com/jetstack/cert-manager/releases/download/v0.8.1/cert-manager.yaml": no matches for kind "Certificate" in version "certmanager.k8s.io/v1alpha1"
unable to recognize "https://github.com/jetstack/cert-manager/releases/download/v0.8.1/cert-manager.yaml": no matches for kind "Issuer" in version "certmanager.k8s.io/v1alpha1"
unable to recognize "https://github.com/jetstack/cert-manager/releases/download/v0.8.1/cert-manager.yaml": no matches for kind "Certificate" in version "certmanager.k8s.io/v1alpha1"
```
- Check pod status
```
kubectl get po -n cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-68cfd787b6-rbbkt              1/1     Running   1          6d21h
cert-manager-cainjector-5975fd64c5-csd9w   1/1     Running   29         6d21h
cert-manager-webhook-5c7f95fd44-qdplg      1/1     Running   0          6d21h
```
### Cert setting
证书管理工具[cert-manager](https://github.com/jetstack/cert-manager),下面的demo我将部署一个webhook到default namespace为例。
- Generate a signing key pair
```
openssl genrsa -out ca.key 2048
```
- Create a self signed Certificate, valid for 10yrs with the 'signing' option set
```
openssl req -x509 -new -nodes -key ca.key -subj "/CN=webhook.default.svc" -days 3650 -reqexts v3_req -extensions v3_ca -out ca.crt
```
### Prepare
- Save the signing key pair as a Secret
```
kubectl create secret tls ca-key-pair \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=default
```
- Creating an Issuer referencing the Secret `issuer.yaml`
```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: default
spec:
  ca:
    secretName: ca-key-pair
```
注:如果想所有namespace都可以使用的话可以参考ClusterIssuer

- Create Certificate `certificate.yaml`
```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
  commonName: webhook.default.svc
  organization:
  - Example CA
  dnsNames:
  - webhook.default.svc
```
- Create ValidatingWebhookConfiguration `webhookconfig.yaml`
```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: webhook
webhooks:
- name: webhook.default.svc
  rules:
  - apiGroups:
    - "*"
    apiVersions:
    - "*"
    operations:
    - CREATE
    resources:
    - pods
  clientConfig:
    service:
      namespace: default
      name: webhook
      path: /pods
    caBundle: <defaultnamespace.secrets.example-com-tls.ca.crt>
```
注: caBundle怎么来的可能很多人不清楚,我特意加个备注吧。secrets.example-com-tls.ca.crt(来自于secrets为example-com-tls的ca.crt字段)。
- webhook deployment `webhook-demo.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webhook
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: webhook
  namespace: default
subjects:
- kind: ServiceAccount
  name: webhook
  namespace: default
roleRef:
  kind: ClusterRole
  name: webhook
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: webhook
  namespace: default
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: webhook
  name: webhook
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/mojo/wehbook:0.0.1
        imagePullPolicy: Always
        name: webhook
        volumeMounts:
        - mountPath: /etc/certs
          name: config
      serviceAccount: webhook
      volumes:
      - name: config
        secret:
          secretName: example-com-tls
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: webhook
  name: webhook
  namespace: default
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app: webhook
```
### Deploy
apply kubernetes yamls
```
1. kubectl apply -f issuer.yaml
2. kubectl apply -f certificate.yaml
3. kubectl apply -f webhookconfig.yaml #记得替换caBundle
4. kubectl apply -f webhook-demo.yaml
```
### validate
`webhook-demo.yaml`只会在pod创建时打印node数量日志,如果执行一下命令你看到相关日志代表已经成功了。
```
kubectl create deploy test-pod-create --image=ngingx
```

### Review cert
对于不知道cert如何生成的同学可能对生成的证书比较好奇,可以通过如下命令查看生成证书的信息。

查看证书相关信息
1. 查看ca信息
```
openssl x509  -noout -text -in ./ca.crt
```
2. 查看key信息
```
openssl rsa -in ca.key -noout -text
```
3.  查看csr信息
```
openssl req -key server.key -in server.csr -noout -text
```