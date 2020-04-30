---
title: Harbor源码浅析
date: 2018-04-12 17:13:10
categories: math
mathjax: true
tags: registry
---
之前写过一篇Harbor的介绍短文，这篇文章对Harbor源码进行了粗略的解析，希望能对Harbor进行扩展的朋友提供帮助，如果不当之处还望指点。
本文主要从以下几点对Harbor源码做简单解析
- 源码结构
- 环境管理
- 容器管理
- 扩展auth,实现sso功能
- jobservice
### 源码结构 
```
| – api (Harbor提供的外部调用的API)
| – auth (认证模块，目前提供两种方式：数据库和LDAP)
| – controllers (控制器相关代码)
| – common (公用模块,包含数据库映射的模型代码、数据持久层、工具类)
| - jobservice(多个Harbor实例之前复制的任务管理模块)
| – make (生成配置、构建镜像)
| – docs (文档)
| – routers (路由相关代码)
| – notification.go (接受Registry发来的镜像上传或下载等事件)
| – token.go (为Registry提供鉴权服务)
| – static (js、css等文件)
| – vendor (依赖的第三方源码)
| – views (html模版文件)
| – main.go (入口函数)
```
Harbor的工作流程如下图所示
![image](http://ofp4ip1ms.bkt.clouddn.com/harbor.png)

### 环境管理
Harbor各个容器运行所需部分变量是通过写入环境变量来实现的,当然如果你要对源码进行更改也可以加入自己的配置文件。如:增加app.conf配置文件。在harbor源码中,你会发现在harbor根目录下有一个make目录, make目录下定义了一系列的脚本文件prepare、模板文件(make/common/templates)、基础配置文件(harbor.cfg)。

开发者通过执行prepare脚本会生成docker容器所需的文件。生成文件如下。
```
config_dir dir is
./common/config
loaded secret key
Clearing the configuration file: ./common/config/ui/env
Clearing the configuration file: ./common/config/registry/config.yml
Clearing the configuration file: ./common/config/db/env
Clearing the configuration file: ./common/config/jobservice/env
Clearing the configuration file: ./common/config/nginx/nginx.conf
Generated configuration file: ./common/config/nginx/nginx.conf(nginx配置文件)
Generated configuration file: ./common/config/ui/env(运行ui所需的环境变量)
Generated configuration file: ./common/config/registry/config.yml(运行registry容器所需配置)
Generated configuration file: ./common/config/db/env(数据库环境变量)
Generated configuration file: ./common/config/jobservice/env(jobservice环境变量)
Clearing the configuration file: ./common/config/ui/private_key.pem
Clearing the configuration file: ./common/config/registry/root.crt
Generated configuration file: ./common/config/ui/private_key.pem(registry私有文件)
Generated configuration file: ./common/config/registry/root.crt(registry密钥文件)
The configuration files are ready, please use docker-compose to start the service.
```
下面对主要的几个配置文件进行简单介绍一下。
- ngnix.conf(上篇文章提到了harbor支持http和https两种协议,下面以http为例)
```
worker_processes auto;
events {
  worker_connections 1024;
  use epoll;
  multi_accept on;
}
http {
  tcp_nodelay on;
  # this is necessary for us to be able to disable request buffering in all cases
  proxy_http_version 1.1;
  upstream registry {
    server 10.0.0.36:5000;
  }
  upstream ui {
    server 10.0.0.36:8080;
  }
  server {
    listen 80;
    # disable any limits to avoid HTTP 413 for large image uploads
    client_max_body_size 0;
    location / {
      proxy_pass http://ui/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      
      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_buffering off;
      proxy_request_buffering off;
    }

    location /v1/ {
      return 404;
    }
    location /v2/ {
      proxy_pass http://registry/v2/;
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      proxy_set_header X-Forwarded-Proto $scheme;
      
      proxy_buffering off;
      proxy_request_buffering off;
    }

    location /service/ {
      proxy_pass http://ui/service/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      
      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      proxy_set_header X-Forwarded-Proto $scheme;
      
      proxy_buffering off;
      proxy_request_buffering off;
    }
  }
}
```
大家可能注意到上面的upstream registry、upstream ui了。这里的配置代理了registry、harbor ui。当请求通过80端口访问时,如果地址是/*、/server/*地址时,直接转发到ui服务。同样的拦截的到/v2/*将转发到registry服务。


- registry服务的配置config.yml
```
version: 0.1
log:
  level: debug
  fields:
    service: registry
storage:
    cache:
        layerinfo: inmemory
    filesystem:
        rootdirectory: /storage
    maintenance:
        uploadpurging:
            enabled: false
    delete:
        enabled: true
http:
    addr: :5000
    secret: placeholder
    debug:
        addr: localhost:5001
auth:
  token:
    issuer: registry-token-issuer
    realm: http://10.0.0.36:8080/service/token
    rootcertbundle: /etc/registry/root.crt
    service: token-service

notifications:
  endpoints:
      - name: harbor
        disabled: false
        url: http://10.0.0.36:8080/service/notifications
        timeout: 3000ms
        threshold: 5
        backoff: 1s

```
上面的核心配置为auth部分,auth中指定了token验证服务地址-->http://10.0.0.36:8080/service/token。 另外notifications部分的配置用于监听registry的相关事件,如：docker push 、docker pull等等。docker deaman会把相关的image的信息都传递到指定的API。这样用户便可以把相关的数据同步到自己的数据库中。 Harbor的token验证服务和docker事件通知监听都设计在harbor的ui中。分配对应src/ui/service/token.go、src/ui/service/notification.go文件。有兴趣
- ui的配置env文件
```

MYSQL_HOST=mysql
MYSQL_PORT=3306
MYSQL_USR=root
MYSQL_PWD=root123
REGISTRY_URL=http://registry:5000
UI_URL=http://ui
CONFIG_PATH=/etc/ui/app.conf
HARBOR_REG_URL=10.0.0.36
HARBOR_ADMIN_PASSWORD=Harbor12345
HARBOR_URL=http://10.0.0.36
AUTH_MODE=db_auth
LDAP_URL=ldaps://ldap.mydomain.com
LDAP_SEARCH_DN=
LDAP_SEARCH_PWD=
LDAP_BASE_DN=ou=people,dc=mydomain,dc=com
LDAP_FILTER=
LDAP_UID=uid
LDAP_SCOPE=3
UI_SECRET=hZIaQRP3RpTuiw7Y
SECRET_KEY=J7qdtDfl30NbrRw4
SELF_REGISTRATION=on
USE_COMPRESSED_JS=on
LOG_LEVEL=debug
GODEBUG=netdns=cgo
EXT_ENDPOINT=http://10.0.0.36
TOKEN_URL=http://ui
VERIFY_REMOTE_CERT=on
TOKEN_EXPIRATION=30
```
ui的env配置位于make/common/config/ui下。这里的变量浅显易懂我就不多介绍了,jobservice同样的处理。

- 容器管理
如基于Harbor源码打包成最终的Docker镜像。请看下面的几个例子。
1. ngnix镜像文件。(最简单的Dockerfile)
```
FROM library/nginx:1.11.5

MAINTAINER mojo_ma@wise2c.com

COPY make/common/config/nginx /etc/nginx
```
把执行prepare生成的ngnix相关配置拷贝到镜像中,并基于ngnix:1.11.5做成的ngnix镜像
2. ui镜像文件
```
FROM golang:1.6.2

MAINTAINER jiangd@vmware.com

RUN apt-get update \
    && apt-get install -y libldap2-dev \
    && rm -r /var/lib/apt/lists/*

COPY src/. /go/src/github.com/vmware/harbor/src
COPY make/common/config/ui/private_key.pem /etc/ui/private_key.pem
WORKDIR /go/src/github.com/vmware/harbor/src/ui
RUN go build -v -a -o /go/bin/harbor_ui

COPY src/ui/views /go/bin/views
COPY src/ui/static /go/bin/static
COPY src/ui/conf  /go/bin/conf
COPY src/favicon.ico /go/bin/favicon.ico
COPY make/jsminify.sh /tmp/jsminify.sh

RUN chmod u+x /go/bin/harbor_ui \
    && sed -i 's/TLS_CACERT/#TLS_CAERT/g' /etc/ldap/ldap.conf \
    && sed -i '$a\TLS_REQCERT allow' /etc/ldap/ldap.conf \
    && /tmp/jsminify.sh /go/bin/views/sections/script-include.htm /go/bin/static/resources/js/harbor.app.min.js /go/bin/

WORKDIR /go/bin/
ENTRYPOINT ["/go/bin/harbor_ui"]
```
这个文件稍微复杂一点。只是步骤多了一点而已。本镜像的基础镜像为golang:1.6.2。
1. RUN apt-get update \
    && apt-get install -y libldap2-dev \
    && rm -r /var/lib/apt/lists/* 更新系统 
2. COPY src/. /go/src/github.com/vmware/harbor/src 拷贝源码到镜像仓库指定目录
3. COPY make/common/config/ui/private_key.pem /etc/ui/private_key.pem 拷贝私钥文件到镜像中（）
4. WORKDIR /go/src/github.com/vmware/harbor/src/ui设定docker的工作目录
5. RUN go build -v -a -o /go/bin/harbor_ui 根据源码构建可执行文件
6. COPY src/ui/views /go/bin/views  
COPY src/ui/static /go/bin/static  
COPY src/ui/conf  /go/bin/conf  
COPY src/favicon.ico /go/bin/favicon.ico  
COPY make/jsminify.sh /tmp/jsminify.sh  拷贝相应的静态资源到容器。
7. WORKDIR /go/bin/
ENTRYPOINT ["/go/bin/harbor_ui"] 指定工作目录并执行生成的可执行文件

根据上面定义的一系列Dockerfile文件,执行make/dev下的docker-compose build 就会打包最终的docker image。进而通过docker-compose up -d、docker-compose down来管理容器的起停。

- 扩展auth,实现sso功能
Harbor中认证模块做了抽象处理,只要实现了指定的接口,并在配置文件中指定对应的认证模式便可以轻松实现自己新增认证需求。下面以我们项目实现的sso认证为例。

默认情况下Harbor提供了db_auth、ldap_auth。为了增加sso功能。我们新加了一个sso_auth。这些auth逻辑都在src/ui/auth目录下。
下面是实现sso_auth的部分代码
```

type Auth struct{}

//实现Authenticate方法
func (auth *Auth) Authenticate(m models.AuthModel) (*models.User, error) {
    //Principal is the sso samlart
    query := soap.BuildSamlValidateQuery(m.Principal)
    url := fmt.Sprintf("%s/samlValidate?TARGET=%s/callback", loader.CAS_SERVER, loader.HARBOR_URL)
    envelope, err := soap.GetSoapEnvelope(url, query)
    if err != nil {
        log.Error("Fail to get CAS info:" + err.Error())
        return &models.User{}, errors.New("Failed to get CAS info")
    }
    var user *models.User
    count, err := dao.GetKeystoneUserCount()
    if err != nil {
        log.Errorf("get account occured error %v", err)
        return user, errors.New("get account from db failed!")
    }
    user, err = getUserFromSoapEnvelope(envelope)
    return user, err
}

// 解析sso获取到的用户 并添加到数据库
func getUserFromSoapEnvelope(envelope *soap.SoapEnvelope) (*models.User, error) {
    username := envelope.Body.Response.Assertion.AttributeStatement.Subject.UserName
    var user *models.User
    var externalID string
    var tenantID string
    var tenantName string
    attributes := envelope.Body.Response.Assertion.AttributeStatement.Attributes
    for _, attribute := range attributes {
        switch attribute.Key {
        case "uid":
            externalID = attribute.Value
        case "project_id":
            tenantID = attribute.Value
        case "project_name":
            tenantName = attribute.Value
        }
    }

    var err error
    newUser := models.User{Username: username, TenantName: tenantName, TenantId: tenantID}
    if user, err = dao.GetUserByKeystone(externalID); err == orm.ErrNoRows { 
        newUser.KeystoneId = externalID
        dao.Register(newUser)
        return &newUser, err
    }

    return user, err
}

// 注册sso_auth
func init() {
    auth.Register("sso_auth", &Auth{})
}
```
另外看一下Harbor的登录逻辑代码
```
var authMode = loader.AUTH_MODE //读取配置确认配置的是那种认证模式
    if authMode == "" {
        authMode = "db_auth"
    }
    log.Debug("Current AUTH_MODE is ", authMode)
    authenticator, ok := registry[authMode] //取出之前auth.Register的auth对象。
    if !ok {
        return nil, fmt.Errorf("Unrecognized auth_mode: %s", authMode)
    }
    if lock.IsLocked(m.Principal) {
        log.Debugf("%s is locked due to login failure, login failed", m.Principal)
        return nil, nil
    }
    user, err := authenticator.Authenticate(m) //执行认证逻辑
    if user == nil && err == nil {
        log.Debugf("Login failed, locking %s, and sleep for %v", m.Principal, frozenTime)
        lock.Lock(m.Principal)
        time.Sleep(frozenTime)
    }
    return user, err
```
就这么简单，一个新增的认证模块就完成了。
- jobservice

jobservice只做了Harbor多个实例进行复制的任务处理逻辑。jobservice对外只提供了复制的接口，仅供ui模块使用。针对jobservice执行的job运行状态主要包括以下几种pending,running,retry,error,stopped,finished,canceled。pending。当启动jobservice服务时候系统会查询数据库中retry状态的job。并重新启动job。如果job一直未完成,每隔一段时间retry状态的job就会尝试继续向目标机上同步镜像。当然运行中的job也可以通过发送stop类型的action给bservice来停止job。

以上是我对Harbor的个人理解。如果有什么不妥之处,还望多多吐槽。