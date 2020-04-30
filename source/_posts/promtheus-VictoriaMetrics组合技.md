---
title: promtheus & VictoriaMetrics组合技
date: 2019-06-26 16:40:13
tags: prometheus
---
目前Prometheus算的上是比较主流的监控和告警系统了,然而原生的Prometheus并没有提供高可用方案,因此基于Prometheus的高可用推出了很多项目, eg: [Prometheus Operator](https://github.com/coreos/prometheus-operator)、[VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics)。下面我要介绍的正是VictoriaMetrics高可用模式。

### VictoriaMetrics是什么
VictoriaMetrics是一个支持高可用、消耗极低、可伸缩的时序数据库,可用于Prometheus监控数据做长期远程存储。

### 为什么要使用VictoriaMetrics
VictoriaMetrics不仅仅是时序数据库,它的优势主要体现在一下几点。
- 对外支持Prometheus相关的API，所以它可以直接用于Grafana作为Prometheus数据源使用, 同时扩展了PromQL, 详细使用可参考 https://github.com/VictoriaMetrics/VictoriaMetrics/wiki/ExtendedPromQL。
- 针对Prometheus的Metrics插入查询具备高性能和良好的扩展性。甚至性能比InfluxDB和TimescaleDB高出20x
- 内存占用方面也做出了优化, 比InfluxDB少10x
- 高性能的数据压缩方式,使存入存储的数据量比TimescaleDB多达70x
- 优化了高延迟IO和低iops的存储
- 操作简单
- 支持从第三方时序数据库获取数据源
- 异常关闭情况下可以保护存储数据损坏

### 准备工作
下面的kubernetes,docker仅仅列了一下我当前测试的版本而已,并不要求强制使用对应版本。
- kubernetes环境(version:v1.13.6)
- docker (version:1.13.1)
- 准备两个VictoriaMetrics实例,用于高可用测试

### 开启VictoriaMetrics、Prometheus组合模式
下面我直接把Prometheus、Grafana部署在一起,为了测试VictoriaMetrics的高可用,我将会以docker的形式启动多实例VictoriaMetrics。

- 以`docker-compose`方式启动VictoriaMetrics多实例
```
version: '3'
services:
  victoria1:
    image: victoriametrics/victoria-metrics:v1.20.1
    ports:
      - 8428:8428
    volumes:
      - /root/victoria1-data:/victoria-metrics-data
  victoria2:
    image: victoriametrics/victoria-metrics:v1.20.1
    ports:
      - 8429:8428
    volumes:
      - /root/victoria2-data:/victoria-metrics-data
```
注意:实例victoria1, victoria2并没有将数据mount到同一个目录,方便后面测试promxy聚合功能
- 定义Prometheus所需的权限信息
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  namespace: default
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
roleRef:
  kind: ClusterRole
  name: prometheus
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: default
```
- 部署Prometheus + Grafana

下面Prometheus配置了kubelet(包括kubelet本身和cadvisor)和一些带有`prometheus.io/scrape: "true"`annotations的采集对象。对于Prometheus配置不熟悉的朋友可以移步 https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s
      evaluation_interval: 15s
      external_labels:
        datacenter: "victoria-1"

    remote_write:
      - url: "http://10.0.0.41:8428/api/v1/write"
        queue_config:
          max_samples_per_send: 10000
      - url: "http://10.0.0.41:8429/api/v1/write"
        queue_config:
          max_samples_per_send: 10000
    scrape_configs:
    - job_name: 'kubelet'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'exporters'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: http
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_endpoints_name]
        target_label: job
      - source_labels: [__meta_kubernetes_service_annotationpresent_prometheus_io_scrape]
        regex: true
        action: keep
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus
  name: prometheus
spec:
  ports:
  - name: prometheus
    port: 9090
    nodePort: 30001
    protocol: TCP
    targetPort: 9090
  - name: grafana
    port: 3000
    nodePort: 30003
    protocol: TCP
    targetPort: 3000
  type: NodePort
  selector:
    app: prometheus
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: prometheus
spec:
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccount: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.10.0
        ports:
        - containerPort: 9090
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        volumeMounts:
        - mountPath: /etc/prometheus
          name: config
      - name: grafana
        image: grafana/grafana:6.2.4
        ports:
        - containerPort: 3000
      volumes:
      - configMap:
          name: prometheus-config
        name: config
```
### 高可用
高可用在生产环境是一个不可避免的话题,VictoriaMetrics同样考虑到这点。VictoriaMetrics的高可用并不像传统意义上的多实例共享一份数据,而是启动了多个实例并各自管理自己的数据。那么问题来了:
1. 如果对接grafana, 数据源怎么填
2. 如果解决了问题1,对外提供一个数据源地址, 监控展示的数据量会跟VictoriaMetrics实例数相关吗(有几个实例返回几倍的数据)?

带着上面的以为,我们来一起揭秘VictoriaMetrics如何使用高可用。我先配上一张简单的流程图表达我的粗略见解。![](/victoria/fluent.png)

1. 配置Prometheus, remote_write中的两个地址分别代表两个VictoriaMetrics实例
```
remote_write:
  - url: "http://10.0.0.41:8428/api/v1/write"
    queue_config:
      max_samples_per_send: 10000
  - url: "http://10.0.0.41:8429/api/v1/write"
    queue_config:
      max_samples_per_send: 10000
```
2. 配置Proxy, 官网给出的是Promxy作为代理。这里[Promxy](https://github.com/jacksontj/promxy)和[Thanos](https://github.com/improbable-eng/thanos)的聚合功能类似。promxy的相关配置如下:
```
apiVersion: v1
data:
  config.yaml: |
    global:
      evaluation_interval: 5s
      external_labels:
        source: promxy
    # remote_write configuration is used by promxy as its local Appender, meaning all
    # metrics promxy would "write" (not export) would be sent to this. Examples
    # of this include: recording rules, metrics on alerting rules, etc.
    remote_write:
      - url: http://localhost:8083/receive
    ##
    ### Promxy configuration
    ##
    promxy:
      server_groups:
        - static_configs:
            - targets:
              - 10.0.0.41:8428
            - targets:
              - 10.0.0.41:8429
          labels:
            sg: k8s01_30001
          anti_affinity: 10s
kind: ConfigMap
metadata:
  name: promxy-config
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: promxy
  name: promxy
spec:
  ports:
  - name: promxy
    port: 8082
    nodePort: 30004
    protocol: TCP
    targetPort: 8082
  type: NodePort
  selector:
    app: promxy
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: promxy
  name: promxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: promxy
  template:
    metadata:
      labels:
        app: promxy
    spec:
      containers:
      - args:
        - "--config=/etc/promxy/config.yaml"
        - "--web.enable-lifecycle"
        command:
        - "/bin/promxy"
        image: quay.io/jacksontj/promxy:latest
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: "/-/healthy"
            port: 8082
          initialDelaySeconds: 3
        name: promxy
        ports:
        - containerPort: 8082
        readinessProbe:
          httpGet:
            path: "/-/ready"
            port: 8082
          initialDelaySeconds: 3
        volumeMounts:
        - mountPath: "/etc/promxy/"
          name: promxy-config
          readOnly: true
      # container to reload configs on configmap change
      - args:
        - "--volume-dir=/etc/promxy"
        - "--webhook-url=http://localhost:8082/-/reload"
        image: jimmidyson/configmap-reload:v0.1
        name: promxy-server-configmap-reload
        volumeMounts:
        - mountPath: "/etc/promxy/"
          name: promxy-config
          readOnly: true
      volumes:
      - configMap:
          name: promxy-config
        name: promxy-config
```
上面配置中给出了promxy的对外访问地址:http://10.0.0.41:30004,当然这里仅仅只是为了测试,用于正式项目中的时候最好使用kubernetes中的dns: promxy.default.svc:8082,这样更灵活。通过上述两步已经完成了所有工作了。

步骤1为Prometheus指定remote_write endpoints, prometheus会并行的把所有采集到的metrics往所有endpoint中丢,从而完成持久化。

步骤2在promxy中配置promethues数据源(由于VictoriaMetrics本身支持Prometheus的API,这里直接对接VictoriaMetrics即可),promxy对外提供唯一的访问入口, 配置Grafana数据源时直接使用promxy对外的访问地址即可。

### 见证奇迹
上面说了一大堆,现在可以见证奇迹了,各位观众请看截图。
- prometheus metrics效果图 ![](/victoria/prometheus.png)
- promxy效果图 ![](/victoria/promxy.png)
- 添加grfana datasource ![](/victoria/datasource.png)
- 添加测试dashboard ![](/victoria/test-dashboard.png)
- grafana dashboard效果图 ![](/victoria/result.png)


相关部署文件可以在 https://github.com/mojo-zd/docs/tree/master/kubernetes/prometheus/high-availability 自行拷贝