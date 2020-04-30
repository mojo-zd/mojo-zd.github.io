---
title: 通过consul实现prometheus联邦
date: 2018-11-20 18:32:47
tags:
---

prometheus作为优秀的开源监控工具,已经被广泛使用。prometheus提供了联邦功能,目的是为了把各个prometheus节点的数据汇总到中心prometheus,我们暂且叫中心prometheus为Data Center,普通的prometheus节点叫Data Node。为了方便验证prometheus通过consul实现联邦功能,下面以docker的形式进行验证。

### 部署cadvisor、consul文件prepare-docker-compose.yaml
```
version: '3'
services:
  cadvisor:
    container_name: cadvisor
    image: google/cadvisor:latest
    ports:
    - 8080:8080
  consul:
    image: consul
    ports:
      - 8400:8400
      - 8500:8500
      - 8600:53/udp
    command: agent -server -client=0.0.0.0 -dev -node=node0 -bootstrap-expect=1 -data-dir=/tmp/consul
    labels:
      SERVICE_IGNORE: 'true'
  registrator:
    image: gliderlabs/registrator
    depends_on:
      - consul
    volumes:
      - /var/run:/tmp:rw
    command: consul://consul:8500
```
cadvisor用来采集container相关指标, consul用来做服务发现。执行`docker-compose -f prepare-docker-compose.yaml up -d`

### 添加Data Node、Data Center

```
version: '3'
services:
  prometheus:
    container_name: prometheus
    image: prom/prometheus
    volumes:
    - ./prometheus/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
    - 9090:9090
  consul-prometheus-federation:
    container_name: consul-prometheus-federation
    image: prom/prometheus
    volumes:
    - ./consul-federation/prometheus:/etc/prometheus/
    ports:
    - 9092:9090
```

#### Data Node prometheus.yml
```
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  # external_labels:
  #     monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
# rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'cadvisor'

    #scheme defaults to 'http'.
    #metrics_path: '/federate'
    static_configs:
      - targets: ['10.0.0.131:8080']
```

#### Data Center promtheus.yml
```
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  # external_labels:
  #     monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
# rule_files:
  # - "first.rules"
  # - "second.rules"

#remote_write:
#    - url: "http://10.0.0.103:9201/write"
#remote_read:
#    - url: "http://10.0.0.103:9201/read"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  #- job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    #static_configs:
    #  - targets: ['localhost:9090']

  - job_name: 'consul-prometheus'
    metrics_path: '/federate'
    honor_labels: true
    params:
      'match[]':
      - '{job="cadvisor"}'

    consul_sd_configs:
      - server: 10.0.0.131:8500
        services: ["prometheus"]
```
启动Data Node、Data Center `docker-compose up -d`

### 验证
访问 Date Node节点 `http://10.0.0.131:9090/graph`效果如下
![image](http://mojo-notes.oss-cn-beijing.aliyuncs.com/federate1.png)

访问 Data Center节点 `http://10.0.0.131:9092/graph`。

target如下: ![image](http://mojo-notes.oss-cn-beijing.aliyuncs.com/federate2.png)

数据采集效果图:
![image](http://mojo-notes.oss-cn-beijing.aliyuncs.com/federate3.png)