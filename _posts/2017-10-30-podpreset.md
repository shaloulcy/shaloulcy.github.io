---
layout: post
author: shalou
title: "kubernetes中的PodPreset"
category: 容器技术
tag: [kubernetes, PodPreset]
---

kubernetes中的PodPreset也是一类API资源，用来给Pod注入secret、environment variables、volumeMounts、volumes等。

PodPreset实在admission的时候起作用的。kubernetes提供了admission机制，对用户对资源的更新操作进行准入规则审计，更新操作包括Create、Update、Delete等、

PodPreset就是作为一个admission controller提供功能的，而且只在创建Pod的时候起作用

PodPreset只对Pod产生作用，通过selector选择特定的Pod添加PodPreset定义提供的资源

<!-- more -->

## 1. 创建一个Pod

```golang
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: ecorp/website
      ports:
        - containerPort: 80
```

Pod的label为app:websit和role:frontend

## 2. 创建PodPreset

```golang
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
  namespace: myns
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

这个PodPreset的意思是，它会给所有标签为role: frontend的Pod注入一个叫cache-volume的volume，给Pod内的所有容器都注入一个DB_PORT的环境边和和一个cache-volume的volumeMounts

## 3. 查看Pod

```golang
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
  annotations:
    podpreset.admission.kubernetes.io/podpreset-allow-database: "resource version"
spec:
  containers:
    - name: website
      image: ecorp/website
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
      ports:
        - containerPort: 80
      env:
        - name: DB_PORT
          value: "6379"
  volumes:
    - name: cache-volume
      emptyDir: {}
```
