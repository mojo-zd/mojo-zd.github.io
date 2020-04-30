---
title: k8s中集成prometheus-operator支持consul形式的prometheus联邦
date: 2018-12-04 17:31:28
tags:
---

### prometheus注册consul
prometheus.yaml
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: prometheus
    app: prometheus-server
    com.wise2c.service: prometheus
    com.wise2c.stack: wisecloud-agent
  name: prometheus
  namespace: wisecloud-agent
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: prometheus
        com.wise2c.service: prometheus
        com.wise2c.stack: wisecloud-agent
        app: prometheus-server
    spec:
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: io.wise2c.service.prometheus
                operator: Exists
      serviceAccountName: wisecloud-agent
      containers:
      - name: service-register
        image: registry.cn-hangzhou.aliyuncs.com/tder/service-register:latest
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: agent-config
        env:
          - name: LISTEN_PORT
            value: "8089"
          - name: SERVICE_NAME
            value: wisecloud-agent-prometheus
          - name: SERVICE_PORT
            value: "9090"
          - name: SERVICE_HEALTH_CHECK_PATH
            value: "/health"
      - name: prometheus
        image: prom/prometheus:v2.3.1
        imagePullPolicy: IfNotPresent
        command:
        - "/bin/prometheus"
        args:
        - "-config.file=/etc/prometheus/prometheus.yml"
        - "-storage.local.path=/prometheus"
        - "-storage.local.retention=180h"
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
```
**注意:** service-register用来注册prometheus到指定的consul服务器

环境变量 | 变量说明
---|---
LISTEN_PORT | service-register运行端口
SERVICE_NAME | 注册到consul的名称
SERVICE_PORT | 注册到consul的端口(这里为prometheus运行端口)
SERVICE_HEALTH_CHECK_PATH | 改地址由service-register提供

prometheus config配置
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: wisecloud-agent
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 30s
      evaluation_interval: 30s
      external_labels:
        environment: k8s
    rule_files:
      - "/etc/prometheus-rules/*.rules"
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']

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

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

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
### 设置prometheus-operator对consul的支持
operator为了提供灵活的prometheus配置。除了提供ServiceMonitor,还提供了`additionalScrapeConfigs`属性。对于consul的支持ServiceMonitor并不能实现。基于`additionalScrapeConfigs`的配置如下(`additionalScrapeConfigs`需要以secrets的形式存在于k8s中)。additional.yaml如下:
```
apiVersion: v1
data:
  prometheus-additional.yaml: LSBqb2JfbmFtZTogJ2ZlZGVyYXRlJwogIHNjcmFwZV9pbnRlcnZhbDogMTVzCgogIGhvbm9yX2xhYmVsczogdHJ1ZQogIG1ldHJpY3NfcGF0aDogJy9mZWRlcmF0ZScKCiAgcGFyYW1zOgogICAgJ21hdGNoW10nOgogICAgLSAne2pvYj0icHJvbWV0aGV1cyJ9JwogICAgLSAne2pvYj0iY2Fkdmlzb3IifScKICAgIC0gJ3tqb2I9Im5vZGUtZXhwb3J0ZXIifScKCiAgY29uc3VsX3NkX2NvbmZpZ3M6CiAgICAgIC0gc2VydmVyOiAxMTEuMjMxLjIxNi43NDo4NTAwCiAgICAgICAgc2VydmljZXM6IFsid2lzZWNsb3VkLWFnZW50LXByb21ldGhldXMtMSJdCg==
kind: Secret
metadata:
  labels:
    managed-by: prometheus-operator
  name: prometheus-k8s-new
  namespace: monitoring
```
prometheus-prometheus.yaml需要做下改动:
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
  baseImage: 192.168.5.14/library/prometheus
  nodeSelector:
    beta.kubernetes.io/os: linux
  replicas: 1
  resources:
    requests:
      memory: 100Mi
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.3.1
  additionalScrapeConfigs:
    name: prometheus-k8s-new
    key: prometheus-additional.yaml
```

```
// 部署
kubectl apply -f additional.yaml prometheus-prometheus.yaml -n monitoring
```