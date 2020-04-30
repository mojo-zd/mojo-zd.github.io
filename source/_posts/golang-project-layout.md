---
title: golang project layout
date: 2018-06-11 17:32:10
tags: golang
---

### 概述

在go项目的开发中很多developer并没有真正关心项目的layout规范。在最近维护的代码中体会颇深。项目结构混乱,文件随意定义,维护起来相当痛苦。为项目维护增加了很高的维护成本。

### Go Directories声明

#### /cmd
项目的主程序入口,每个应用程序的目录名称应与你想要执行文件的名称相匹配(eg:/cmd/myapp)。

最好不要把太多代码放到该目录下,如果你觉得你的代码需要被其他模块引用或者被其他项目引用你应该把这类代码放到/pkg目录下。如果你的代码不会被重复使用也不想被其他项目引用,那么你应该将这类代码放到/internal目录。

/cmd目录使用可以参考一下这些[示例项目](https://github.com/golang-standards/project-layout/blob/master/cmd/README.md)

#### /internal
该目录主要存放项目的私有代码,这些代码不会被其他应用程序或者库所引用。正常来讲项目的实际代码应该放在/internal/app目录下(eg:/internal/app/myapp)。该目录下共享的代码应该放在/internal/pkg下(eg:/internal/pkg/myprivlib)。

#### /pkg
此目录下的代码可以被其他应用程序导入使用(eg:/pkg/mypubliclib)。/pkg[示例项目](https://github.com/golang-standards/project-layout/blob/master/pkg/README.md)

#### /vendor
应用依赖(通过你喜欢的项目依赖管理工具来管理),如果你是构建一个通用的libaray建议不要提交相应的应用依赖。

### Service Application Directories
#### /api
OpenAPI/Swagger规范, Json schema文件, protocol定义文件
参考[示例](https://github.com/golang-standards/project-layout/blob/master/api/README.md)

### Web Application Directories
#### /web
Web项目特定组件 eg:static web assets, server side templates and SPAs

### Common Application Directories
#### /configs
配置模板或者默认配置

#### /init
System init (systemd, upstart, sysv) and process manager/supervisor (runit, supervisord) configs.

#### /scripts
执行各种脚本,如 build, install, analysis, etc operations

像Makefile这种应该放在根目录下(eg:https://github.com/hashicorp/terraform/blob/master/Makefile).

/scripts相关[示例](https://github.com/golang-standards/project-layout/blob/master/scripts/README.md)

#### /build
打包&CI

把例如cloud(AMI), container(Docker),OS(deb, rpm, pkg) 打包配置和脚本放在 /build/pakcage目录 

把CI(travis, circle, drone)配置和脚本放到/build/ci目录

#### /deployments
IaaS, PaaS, system and container orchestration deployment configurations and templates (docker-compose, kubernetes/helm, mesos, terraform, bosh).

### /test
Additional external test apps and test data. Feel free to structure the /test directory anyway you want. For bigger projects it makes sense to have a data subdirectory (e.g., /test/data).

See the [/test](https://github.com/golang-standards/project-layout/blob/master/test/README.md) directory for examples

### Other Directories
#### /docs
设计&用户文档,另外你的godoc生成的目录也是放在该目录下

/docs [示例](https://github.com/golang-standards/project-layout/blob/master/docs/README.md)

#### /examples
项目或者公共库示例
/examples [示例](https://github.com/golang-standards/project-layout/blob/master/examples/README.md)

#### /third_party
外部工具