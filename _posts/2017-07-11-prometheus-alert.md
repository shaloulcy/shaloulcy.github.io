---
layout: post 
author: shalou
title:  "prometheus报警配置"   
category: 容器技术
tag: ["prometheus"]
---

在CloudDB中，我们使用Prometheus对容器进行监控和报警，本文主要介绍报警功能，报警功能主要由两部分构成，一部分是Prometheus的报警规则，另一部则由AlertManager通过WebHook或email的形式通知用户。

## 1. Prometheus报警配置
Prometheus的配置文件为prometheus.yaml，Prometheus启动时可以通过-config.file参数制定其配置文件，配置文件中与报警相关的配置选项如下

<!-- more -->

```
global:
  evaluation_interval: 30s   //定时执行报警规则的间隔，警报将发送给alertmanager
rule_files:
- /etc/prometheus/rules/rules-0/*.rules   //报警规则文件
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

其中evaluation_interval指定了定时执行报警规则的间隔，rule_files指定了报警规则文件所在位置，alerting主要配置向谁报警，这里我们将警报发送给alertmanager，由于alertmanager安装在k8s集群中，因此我们这里用到了alertmanager的发现机制，当然也通过alertmanager.url静态配置其URL

## 2. 报警规则(alert rules)

想要Prometheus进行报警，需要我们自己配置报警规则，在k8s环境下，所有的报警配置都配置在configmap中，当报警规则发现变化时，会有组件重新下载规则文件，然后通知Prometheus Reload。通过Prometheus Operator安装完Prometheus后，已经有一些默认的报警规则，但现有的报警规则主要针对物理节点以后系统daemon进程报警，假如用户需要针对自己的应用报警，则需要自己添加规则。报警规则语法如下

```
ALERT <alert name>
  IF <expression>
  [ FOR <duration> ]
  [ LABELS <label set> ]
  [ ANNOTATIONS <label set> ]
```

alert name指报警名字

expression主要由Prometheus的查询语法构成

duration表示时间段，若符合expression的时间超过duration，则发送报警

LABELE和ANNOTATIONS会包括在报警信息中

我们查看如下报警规则，该报警规则表示假如实例宕机超过5分钟，则报警

```
ALERT InstanceDown
  IF count(up) == 0
  FOR 5m
  LABELS { severity = "page" }
  ANNOTATIONS {
    summary = "Instance {{ $labels.instance }} down",
    description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.",
  }
```

## 3. AlertManager

Alertmanager处理由类似Prometheus服务器等客户端发来的警报，之后需要删除重复、分组，并将它们通过路由发送到正确的接收器，比如电子邮件、Slack等。Alertmanager还支持沉默和警报抑制的机制。

### 3.1 分组(Grouping)
分组是指当出现问题时，Alertmanager会收到一个单一的通知，而当系统宕机时，很有可能会同时生成成百上千的警报，这种机制在较大的中断中特别有用。

例如，当数十或数百个服务的实例在运行，网络发生故障时，有可能服务实例的一半不可达数据库。在告警规则中配置为每一个服务实例都发送警报的话，那么结果是数百警报被发送至Alertmanager。

但是作为用户只想看到单一的报警页面，同时仍然能够清楚的看到哪些实例受到影响，因此，人们通过配置Alertmanager将警报分组打包，并发送一个相对看起来紧凑的通知。

分组警报、警报时间，以及接收警报的receiver是在配置文件中通过路由树配置的

### 3.2 抑制(Inhibition)

抑制是指当警报发出后，将抑制其他相关警报的发送。

例如，当警报被触发，通知整个集群不可达，可以配置Alertmanager忽略由该警报触发而产生的所有其他警报，这可以防止通知数百或数千与此问题不相关的其他警报。

抑制机制可以通过Alertmanager的配置文件来配置。

### 3.3 沉默(Silences)
沉默是一种简单的特定时间静音提醒的机制。一种沉默是通过匹配器来配置，就像路由树一样。传入的警报会匹配RE，如果匹配，将不会为此警报发送通知。

沉默机制可以通过Alertmanager的Web页面进行配置。

### 3.4 AlertManager的配置
AlertManager通过alertmanager.yaml文件配置，AlertManager启动时通过-config.file参数进行配置，目前该文件是以secret的形式挂载到容器内部的，更新secret时，AlertManger会动态加载

alertmanager.yaml配置文件如下

```
global:
  resolve_timeout: 5m
  smtp_from: 'cyli@sysnew.com'
  smtp_smarthost: '172.17.249.182:25'
  smtp_smarthost: 'cyli@sysnew.com'
  smtp_auth_password: 'xxxxx'
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

文件中的global是全局生效的部分，route部分配置路由，根据警报的名称、label等路由到不同的router,router中有一个receiver，即将警报的接收者。最下面有一个总的receivers，里面定义了所有的receiver。

通知目前支持email、webhook、hipchat等形式，配置文件的以email为例，相关的配置选项是smtp_from、smtp_smarthost、smtp_smarthost、smtp_auth_password、smtp_require_tls。

group选项是用来支持上面提到的分组概念，属于同一分组的警报将一块发送。

group_wait指分组创建多久后才可以发送压缩的警报，也就是初次发警报的延时

group_interval，当有新的警报加入group时，隔多久发送压缩的警报

repeat_interval 重复发送压缩的警报的时间间隔

