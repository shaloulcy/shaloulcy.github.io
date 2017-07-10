---
layout: post 
author: shalou
title:  "prometheus and alertmanager configuration"   
category: 容器技术
tag: ["prometheus"]
---

Prometheus配置文件如下

<!-- more -->

```golang
global:
  evaluation_interval: 30s
  scrape_interval: 30s
  external_labels: {}
rule_files:
- /etc/prometheus/rules/rules-0/*.rules
scrape_configs:
- job_name: monitoring/alertmanager/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_alertmanager
    regex: main
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: monitoring
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: web
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - target_label: endpoint
    replacement: web
- job_name: monitoring/kube-apiserver/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  scheme: https
  tls_config:
    insecure_skip_verify: false
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    server_name: kubernetes
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_component
    regex: apiserver
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_provider
    regex: kubernetes
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: default
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: https
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_component
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: https
- job_name: monitoring/kube-controller-manager/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    regex: kube-controller-manager
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: kube-system
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: http-metrics
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_k8s_app
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: http-metrics
- job_name: monitoring/kube-scheduler/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    regex: kube-scheduler
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: kube-system
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: http-metrics
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_k8s_app
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: http-metrics
- job_name: monitoring/kube-state-metrics/0
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    regex: kube-state-metrics
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: monitoring
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: http-metrics
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_k8s_app
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: http-metrics
- job_name: monitoring/kubelet/0
  honor_labels: true
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    regex: kubelet
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: kube-system
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: http-metrics
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_k8s_app
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: http-metrics
- job_name: monitoring/node-exporter/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    regex: node-exporter
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: monitoring
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: http-metrics
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - source_labels:
    - __meta_kubernetes_service_label_k8s_app
    target_label: job
    regex: (.+)
    replacement: ${1}
  - target_label: endpoint
    replacement: http-metrics
- job_name: monitoring/prometheus/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  scrape_interval: 30s
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_prometheus
    regex: k8s
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: monitoring
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: web
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - target_label: endpoint
    replacement: web
- job_name: monitoring/prometheus-operator/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_k8s_app
    regex: prometheus-operator
  - action: keep
    source_labels:
    - __meta_kubernetes_namespace
    regex: monitoring
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: http
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: job
    replacement: ${1}
  - target_label: endpoint
    replacement: http
alerting:
  alertmanagers:
  - kubernetes_sd_configs:
    - role: endpoints
    scheme: http
    relabel_configs:
    - action: keep
      source_labels:
      - __meta_kubernetes_service_name
      regex: alertmanager-main
    - action: keep
      source_labels:
      - __meta_kubernetes_namespace
      regex: monitoring
    - action: keep
      source_labels:
      - __meta_kubernetes_endpoint_port_name
      regex: web
```

alertmanager配置文件如下

```
global:
  resolve_timeout: 5m
  smtp_from: 'cyli@sysnew.com'
  smtp_smarthost: '172.17.249.182:25'
  smtp_auth_username: 'cyli@sysnew.com'
  smtp_auth_password: 'xxxxxx'
  smtp_require_tls: false
route:
  group_by: ['job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'cyli'
  routes:
  - match:
      alertname: DeadMansSwitch
    receiver: 'cyli'
receivers:
- name: 'cyli'
  email_configs:
  - to: 'cyli@sysnew.com'
```
