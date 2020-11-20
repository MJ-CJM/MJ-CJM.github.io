---
title: Kubernetes核心数据结构1.0
date: 2020-11-14 13:38:35
tags: Kubernetes
categories: Kubernetes源码解析
---

## Group、Version、Resource核心数据结构

在整个 Kubernetes 体系架构中，资源是 Kubernetes 最重要的概念，它本质上是一个资源控制系统————注册、管理、调度资源并维护资源的状态。

Kubernetes 将资源进行分组和版本化，形成 Group、Version、Resource。具体机构图如下：

![kubernetes核心数据结构图](../image/kubernetes核心数据结构.jpg)