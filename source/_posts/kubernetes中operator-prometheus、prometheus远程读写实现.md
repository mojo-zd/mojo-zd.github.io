---
title: kubernetes中operator-prometheus、prometheus远程读写实现
date: 2018-10-23 16:28:48
tags:
---
### remote read、write优点
- 长期持久化监控数据
- 集中管理多个prometheus中的监控数据
- 对外提供统一的监控数据出口

### 环境准备
准备两套k8s集群, 分别部署prometheus prometheus-operator
docker server version | kubernetes version
---|---
1.13.1 | v1.8.6

### 启动监控数据database
以下以opentsdb作为存储prometheus监控数据的database。注:opentsdb较高版本对于带有":"的key或value并不支持。
eg:
```
{
      "aggregator": "sum",
      "metric": "cpu",
      "tags": {
        "demo-name": "opentsdb-test",
        "host": "localhost:9090",
        "try-name": "bluebreezecf-sample"
      }
    }
```
为了方便测试,下面以容器的方式启动opentsdb数据库
```
docker run -d -p 4242:4242 -p 8070:8070 stackexchange/bosun:0.6.0-pre
```

### prometheus配置
#### prometheus权限模型定义
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus-colletor
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus-colletor
roleRef:
  kind: ClusterRole
  name: kube-system
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: prometheus-colletor
  namespace: prometheus-colletor

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-colletor
```


#### 定义prometheus配置文件configmap
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-system
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 30s
      evaluation_interval: 30s
    remote_write:
    - url: http://127.0.0.1:9201/write
    remote_read:
    - url: http://127.0.0.1:9201/read
    rule_files:
      - "/etc/prometheus-rules/*.rules"
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    - job_name: 'kubernetes-services'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name
    
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
```

#### prometheus 部署

以下prometheus.yaml文件中包含了prometheus-server、prometheus-adapter。adapter在此功能中担任的角色相当于一个装换器,write的时候将prometheus采集的数据格式装换为database的记录,read的时候再将database记录转换为prometheus类型的数据格式。对于opentsdb官网并没有提供带有remote_read的adapter。为此我们扩展了官网的opentsdb库增加了remote_read功能。
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: prometheus
    app: prometheus-server
  name: prometheus
  namespace: wisecloud-agent
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: prometheus
        app: prometheus-server
    spec:
      serviceAccountName: wisecloud-agent
      containers:
      - name: prometheus
        image: prom/prometheus:v2.3.1
        imagePullPolicy: IfNotPresent
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/"
        - "--storage.tsdb.retention=168h"
        # - "--alertmanager.url=http://127.0.0.1:9093"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: "/prometheus"
        - name: config-volume
          mountPath: "/etc/prometheus"
        - name: alert-roles
          mountPath: "/etc/prometheus-rules"
        - name: local-timezone
          mountPath: "/etc/localtime"
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      - name: prometheus-adapter
        image: registry.cn-hangzhou.aliyuncs.com/tder/prometheus-adapter:v2.3
        ports:
        - containerPort: 9021
        protocol: TCP
        args:
        - -opentsdb-url=http://xx.xx.xx.xx:4242 // xx.xx.xx.xx为database运行主机ip
      #- name: altermanager
      #  image: prom/alertmanager:v0.15.0
      #  imagePullPolicy: IfNotPresent
      #  args:
      #    - '--config.file=/etc/alertmanager/config.yml'
      #    - '--storage.path=/alertmanager'
      #  ports:
      #  - containerPort: 9093
      #    protocol: TCP
      #  volumeMounts:
      #  - name: alert-conf
      #    mountPath: /etc/alertmanager
      #  - name: local-timezone
      #    mountPath: "/etc/localtime"
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
      - name: data
        hostPath:
          path: /prometheus
      - name: alert-roles
        hostPath:
          path: /etc/prometheus-rules
      - name: alert-conf
        hostPath:
          path: /etc/alertmanager
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: local-timezone
        hostPath:
          path: /etc/localtime
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-server
  namespace: kube-system
spec:
  selector:
    app: prometheus-server
  ports:
  - protocol: TCP
    port: 9090
    targetPort: 9090
    nodePort: 32005
  type: NodePort
```

**注:** 高版本的prometheus在配置storage选项的时候会报permission错误,测试可以给对应的目录设置所有人员可读写模式。如上: `chmod 777 /prometheus`

#### 部署
```
kubectl apply -f prometheus.yaml
```

### prometheus operator配置
#### adapter.yaml
```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: prometheus-adapter
  namespace: monitoring
  labels:
    app: prometheus-adapter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-adapter
  template:
    metadata:
      labels:
        app: prometheus-adapter
    spec:
      containers:
      - name: adapter
        image: registry.cn-hangzhou.aliyuncs.com/tder/prometheus-adapter:v2.3
        args:
        - -opentsdb-url=http://yy.yy.yy.yy:4242 //yy.yy.yy.yy为database运行主机ip
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-adapter
  namespace: monitoring
spec:
  selector:
    app: prometheus-adapter
  ports:
  - protocol: TCP
    port: 9201
    targetPort: 9201
    nodePort: 30902
  type: NodePort
```

#### prometheus-operator中prometheus的部署文件添加`remoteRead、remoteWrite`
被修改的文件为[prometheus-prometheus.yaml](https://github.com/coreos/prometheus-operator/blob/master/contrib/kube-prometheus/manifests/prometheus-prometheus.yaml)
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - name: alertmanager-main
      namespace: monitoring
      port: web
  baseImage: quay.io/prometheus/prometheus
  nodeSelector:
    beta.kubernetes.io/os: linux
  replicas: 2
  remoteRead: 
  - url: http://xx.xx.xx.xx:30902/read
  remoteWrite:
  - url: http://xx.xx.xx.xx:30902/write
  resources:
    requests:
      memory: 400Mi
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.3.1
```
**注:** 目前官方提供的opentsdb的remote_write在prometheus-operator中支持有问题,时间长了导致写入数据失败。

#### 部署
```
kubectl apply -f adapter.yaml
kubectl apply -f prometheus-prometheus.yaml
```

### 验证
- write
通过查看adapter日志可以看到是否写入数据到database

- read
访问prometheus提供的ui界面, 默认情况prometheus并没有暴露端口。需要验证的话用户可以自己修改prometheus的service暴露9090对外可访问