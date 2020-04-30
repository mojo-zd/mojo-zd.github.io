---
title: 运行docker容器时如何动态传入golang中的flag参数
date: 2018-06-29 17:05:08
tags: docker
---

容器使用已经有一段时间了,对于运行docker时怎么动态传入golang中的flag参数还没有尝试过。以前使用的时候都是使用环境变量的形式,golang既然有flag的形式,那就有必要研究一下。下面主要针对docker、docker-compose两种方式

### docker file define
```
FROM alpine:3.7

COPY demo /go/bin/demo

WORKDIR /go/bin/

ENTRYPOINT ["/go/bin/demo"]
```

```
//build docker image
docker build -t test .
```


### docker
- 直接运行
```
go run main.go -url=localhost -port 9999

output:

flag url is  localhost
flag port is  9999
```

- docker run 
```
docker run test -url=localhost -port=9999

output:

flag url is  localhost
flag port is  9999
```

### docker-compose
- define docker-compose
```
version: "3"
services:
  demo:
    image: test
    entrypoint:
      - /go/bin/demo
      - -url=localhost
      - -port=9999
```

- run
```
docker-compose up -d
//查看对应容器日志 同样打印了上述信息
```