---
title: client-go 最终章
date: 2020-12-16 22:36:59
tags: [Kubernetes, client-go]
categories: Kubernetes源码解析
---

## EventBroadcaster 事件管理器

Kubernetes 的事件（Event）是一种资源对象（Resource Object），用于展示集群内发生的情况，Kubernetes 系统中的各个组件会将运行发生的各种事件上报给 Kubernetes API Server。
> 此处的 Event 事件是 Kubernetes 所管理的 Event 资源对象，而非 Etcd 集群监控机制产生的回调事件。

* Kubernetes 系统以 Pod 资源为核心，Deployment,StatefulSet,ReplicaSet,DaemonSet,CronJob 等,最终都会创建出 Pod。因此 Kubernetes 事件也是围绕 Pod 进行的，在 Pod 生命周期内的关键步骤都会产生事件消息。

* Event 资源数据结构描述了当前时间段内发生了哪些关键性事件。事件有两种类型，分别为 Normal 和 Warning，前者为正常事件，后者为警告事件。

### EventBroadcaster 事件管理机制运行原理

Actor 可以是 Kubernetes 系统中的任意组件，当组件中发生了一些关键性事件时，可通过 EventRecorder 记录事件。EventBroadcaster 事件管理机制可以分为如下部分：

* EventRecorder: 事件生产者，也称为事件记录器。Kubernetes 系统组件通过 EventRecorder 记录关键性事件。
* EventBroadcaster: 事件消费者，也称为事件广播器。EventBroadcaster 消费 EventRecorder 记录的事件并将其分发给目前所有已连接的 broadcasterWatcher.分发过程由两种机制，分别是非阻塞分发机制和阻塞分发机制。
* broadcasterWatcher: 观察者管理，用于定义事件的处理方式，例如上报事件至 Kubernetes API Server。

#### EventRecorder

EventRecorder 拥有如下 4 种记录方法，EventRecorder 事件记录器接口代码示例如下：

```
type EventRecorder interface {
    Event(...)
    // 对刚发生的事件进行记录
    Eventf(...)
    // 通过使用 fmt.Sprintf 格式化输出事件的格式
    PastEventf(...)
    // 允许自定义事件发生的时间，以记录已经发生过的消息
    AnnotatedEventf(...)
    // 功能与 Eventf 一样，但附加了注释字段
}
```

Event 方法示例，记录当前发生的事件，Event -> recorder.generateEvent -> recorder.Action 代码示例如下：

```
func (m *Broadcaster) Action (action EventType, obj runtime.Object) {
    m.incoming <- Event{action, obj}
}
```

Action 函数通过 goroutine 实现异步操作，该函数将事件写入 m.incommit Chan 中，完成事件生成过程。

#### EventBroadcaster 

EventBroadcaster 消费 EventRecorder 记录的事件并将其分发给目前所有已连接的 broadcasterWatcher.EventBroadcaster 通过 NewBroadcaster 函数进行实例化：

```
func NewBroadcaster() EventBroadcaster {
    return &eventBroadcasterImpl{watch.NewBroadcaster(maxQueuedEvents, watch.DropIfChannelFull), defaultSleepDuration}
}
```

在实例化过程中，会通过 watch.NewBroadcaster 函数在内部启动 goroutine 来监控 m.incoming,并将监控的事件通过 m.distribute 函数分发给所有已连接的 broadcasterWatcher。在非阻塞分发机制下使用 DropIfChannelFull 标识，在阻塞分发机制下使用 WaitIfChannelFull 标识，默认为 DropIfChannelFull 标识。代码示例如下：

```
func (m *Broadcaster) distribute (event Event) {
    m.lock.Lock()
    defer m.lock.Unlock()
    if m.fullChannelBehavior == DropIfChannelFull {
        for _, w := range m.watchers {
            select {
                case w.result <- event:
                case <- w.stopped:
                default:
            }
        }
    } else {
        for _, w := range m.watchers {
            select {
                case w.result <- event:
                case <- w.stopped:
            }
        }
    }
}
```

在分发过程中，DropIfChannelFull 标识位于 select 多路复用中，使用 default 关键字做非阻塞分发，当 w.result 缓冲区满的时候，事件会丢失。WaitIfChannelFull 标识也位于 select 多路复用中，没有 default 关键字，当 w.result 缓冲区满的时候，分发过程会阻塞并等待。

> Kubernetes 中的事件与其他的资源不同，它有一个很重要的特性，那就是它可以丢失。（与 etcd 的性能有关，事件的重要性远低于集群的稳定性）

#### broadcasterWatcher

broadcasterWatcher 是每个 Kubernetes 系统组件自定义处理事件的方式。每个 broadcasterWatcher 拥有两种自定义处理事件的函数，分别介绍如下：

* StartLogging: 将事件写入日志中。
* StartRecordingToSink: 将事件上报至 Kubernetes API Server 并存储至 Etcd 集群。

以 kube-scheduler 组件为例，该组件为一个 broadcasterWatcher，通过 StartLogging 函数将事件输出至 klog stdout 标准输出，通过 StartRecordingToSink 函数将关键性事件上报至 Kubernetes API Server。代码示例如下：

```
if cc.Brodcaster != nil && cc.EventClient != nil {
    cc.Broadcaster.StartLogging(klog.V(6).Infof)
    cc.Broadcaster.StartRecordingToSink(&vlcore.EventSinkImpl{Interface: cc.EventClient.Events("")})
}
```

StartLogging 和 StartRecordingToSink 函数依赖于 StartEventWatcher 函数，该函数内部运行了一个 goroutine，用于不断监控 EventBroadcaster 来发现事件并调用相关函数对事件进行处理。

下面重点介绍下 StartRecordingToSink 函数，kube-scheduler 组件将 v1core.EventSinkImpl 作为上报事件的自定义函数。

* 上报事件有三种方法，分别是 Create(Post), Update(Put), Patch(Patch),以 Create 方法为例，Create -> e.Interface.CreateWithEventNamespace 代码示例如下：

```
func (e *events) CreateWithEventNamespace(event *v1.Event) (*v1.Event, error) {
    ...
    result := &v1.Event{}
    err := e.client.Post().NamespaceIfScoped(event.Namespace, len(event.Namespace) > 0).Resource("events").Body(event).Do().Into(result)
    return result, err
}
```

上报过程通过 RESTClient 发送 Post 请求，将事件发送至 Kubernetes API Server，最终存储在 Etcd 集群中。

## 代码生成器

前面已经讲解过 5 种代码生成器的核心实现，它们用于在构建 Kubernetes 核心组件之前生成执行代码。

本节将介绍关于 client-go 的其他 3 种代码生成器，如下表所示：

代码生成器 | 说明
--- | ---
client-gen | client-gen 是一种为资源生成 ClientSet 客户端的工具
lister-gen | lister-gen 是一种为资源生成 Lister 的工具（即 get 和 list 方法）
informer-gen | informer-gen 是一种为资源生成 informer 的工具

### client-gen 代码生成器

client-gen 代码生成器是一种为资源生成 ClientSet 客户端的工具。ClientSet 对 Group,Version,Resource 进行了封装，可以针对资源执行生成资源操作方法（create,update,delete,get 等），这些方法由 client-gen 代码自动生成器自动生成。

client-gen 代码生成器的代码生成策略与 Kubernetes 的其他代码生成器类似，都通过 Tags 来识别一个包是否需要生成代码及确定生成代码的方式。

* 生成基本的资源操作方法，例如 create, update, delete, get, list, patch, watch 等方法。另外，如果存在 Status 字段，则生成 UpdateStatus 函数，其 Tags 形式如下：

```
// +genclient
```

* 生成基本的资源操作函数，不生成 Namespaced 函数，其 Tags 形式如下：

```
// +genclient:nonNamespaced
```

* 即便存在 Status 字段，也不生成 UpdateStatus 函数，其 Tags 形式如下：

```
// +genclient:noStatus 
```

* 不生成基本的资源操作方法，其 Tags 形式如下：

```
// +genclient:noVerbs
```

* 仅生成指定的基本操作方法，例如 create, get 方法，其 Tags 形式如下：

```
// +genclient:onlyVerbs=create,get
```

* 生成基本的资源操作方法，但排除 watch 方法，其 Tags 形式如下：

```
// +genclient:skipVerbs=watch
```

* 生成相关扩展函数，例如生成 UpdateScale 函数，在 UpdateScale 函数中会对 scale 子资源进行 update(更新)操作，input 和 result 是用于 设置输入和输出的参数，其 Tags 形式如下：

```
// +genclient:method=UpdateScale,verb=update,subresource=scale,input=Scale,result=Scale
```

### lister-gen 代码生成器

lister-gen 代码生成器是一种为资源生成 Lister 的工具。Lister 为每一个 Kubernetes 资源提供 Lister 功能（即提供 get 和 list 方法）。get 和 list 方法为客户端提供只读的本地缓存数据。

* lister-gen 代码生成器的代码生成策略与 Kubernetes 的其他代码生成器类似，都通过 Tags 来识别一个包是否需要生成代码及确定生成代码的方式。

* lister-gen 代码生成器与其他代码生成器相比，其并没有可用的 Tags，它依赖于 client-gen 的代码生成器 // +genclient 标签。

### informer-gen 代码生成器

informer-gen 是一种为资源生成 informer 的工具，Informer 为 client-go 提供了与 Kubernetes API Server 通信机制，这些功能由 informer-gen 代码生成器自动生成。

* informer-gen 代码生成器的代码生成策略与 Kubernetes 的其他代码生成器类似，都通过 Tags 来识别一个包是否需要生成代码及确定生成代码的方式。

* informer-gen 代码生成器与其他代码生成器相比，其并没有可用的 Tags，它依赖于 client-gen 的代码生成器 // +genclient 标签,类似于 lister-gen 代码生成器。