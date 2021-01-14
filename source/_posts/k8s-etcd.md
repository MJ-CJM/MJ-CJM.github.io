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

## RESTStorage 存储服务通用接口

所有通过 RESTful API 对外暴露的资源必须实现 RESTStorage 接口。代码示例如下：

```
type Storage interface {
    New() runtime.Object
}
```

Kubernetes 资源一般通过 NewStorage 函数或 NewREST 函数实例化。以 Deployment 资源为例，代码示例如下：

```
type REST struct {
    *genericregistry.Store
    categories []string
}

type StatusREST struct {
    store *genericregistry.Store
}

type DeploymentStorage struct {
    Deployment *REST
    Status *StatusREST
    ...
}
```

Deployment 资源定义了 REST 数据结构与 StatusREST 数据结构，其中 REST 数据结构用于实现 deployment 资源的 RESTStorage 接口，而 StatusREST 数据结构用于实现 deployment/status 子资源的 RESTStorage 接口，每一个 RESTStorage 接口都对 RegistryStore 操作进行了封装，例如，对 deployment/status 子资源进行 Get 操作时，实际执行的是 RegistryStore 操作，代码示例如下：

```
func (r *StatusREST) Get(ctx context.Context, name string, options *metav1.GetOptions)(runtime.Object, error) {
    return r.store.Get(ctx, name, options)
}
```

## RegistryStore 存储服务通用操作

实现了资源存储的通用操作，例如，在存储资源对象之间或者之后执行某个函数。

RegistryStore 中定义了如下两种函数。

* Before Func：也称 Strategy 预处理，它被定义为在创建对象之前调用，做一些预处理工作。
* After Func：它被定义为在创建资源对象之后调用，做一些收尾工作。

最后，Storage 字段是 RegistryStore 对 Storage.Interface 通用存储接口进行的封装，实现了对 Etcd 集群的读/写操作。以 RegistryStore 的 Create 方法为例，代码示例如下：

```
func (e *Store) Create(...) (runtime.Object, error) {
    // 1. 进行预处理操作
    if err := rest.BeforeCreate(e.CreateStrategy, ctx, obj); err != nil {
        return nil, err
    }
    ...
    // 2. 创建资源对象
    if err := e.Storage.Create(ctx, key, obj, out, ttl, dryrun.IsDryRun(options.DryRun)); err != nil {
        ...
    }
    // 3. 执行收尾操作
    if e.AfterCreate != nil {
        if err := e.AfterCreate(out); err != nil {
            return nil, err
        }
    }
}
```

## Storage.Interface 通用存储接口

实现通用存储接口的分别是 CacherStorage 资源存储对象和 UnderlyingStorage 资源存储对象，分别介绍如下：

* CacherStorage：带有缓存功能的资源存储对象。
* UnderlyingStorage：底层存储对象，真正与 Etcd 集群交互的资源存储对象。

CacherStorage 实际上是在 UnderlyingStorage 之上封装了一层缓存层，在 genericregistry.StorageWithCacher 函数实例化的过程中，也会创建 UnderlyingStorage 底层存储对象。

在 CacherStorage 的实例化过程中，在装饰器函数里实现了 UnderlyingStorage 和 CacherStorage 的实例化过程，代码示例如下：

```
func StorageWithCacher(capacity int) generic.StorageDecorator {
    return func(
        ...
        s, d := generic.NewRawStorage(storageConfig)
        ...
        cacher := cacherstorage.NewCacherFromConfig(cacherConfig)
        ...
        return cacher, destroyFunc
    )
}
```

## CacherStorage 缓存层





