---
title: Etcd 存储核心实现-Etcd 存储架构设计
date: 2020-12-21 18:07:40
tags: [Kubernetes, etcd]
categories: Kubernetes源码解析
---

Etcd 集群存储 Kubernetes 系统的集群状态和元数据，其中包括所有 Kubernetes 资源对象信息，资源对象状态，集群节点信息等。Kubernetes 将所有数据存储至 Etcd 集群前缀为 /registry 的目录下。

## Etcd 存储架构设计

![Etcd 存储架构设计图](../image/Etcd存储架构设计.jpg)

### RESTStorage

实现了 RESTful 风格的对外资源存储服务的 API 接口。

### RegistryStore

实现了资源存储的通用操作，例如，在存储资源对象之间或者之后执行某个函数。

### Storage.Interface 

通用存储接口，该接口定义了资源的操作方法。

### CacherStorage 

带有缓存功能的资源存储对象，它是 Storage.Interface 通用存储接口的实现。CacherStorage 缓存层的设计有利于 Etcd 集群中的数据能够获得快速的响应，并与 Etcd 集群数据保持一致。

### UnderlyingStorage

底层存储，也被称为 BackendStorage（后端存储），是真正与 Etcd 集群交互的资源存储对象，CacherStorage 相当于 UnderlyingStorage 的缓存层。UnderlyingStorage 同样也是 Storage.Interface 通用存储接口的实现。