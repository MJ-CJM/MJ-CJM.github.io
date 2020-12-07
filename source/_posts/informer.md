---
title: client-go2.0(informer)
date: 2020-12-01 21:42:07
tags: [Kubernetes, client-go]
categories: Kubernetes源码解析
---

Kubernetes 的其他组件都是通过 client-go 的 Informer 机制与 Kubernetes API Server 进行通信的，并且保证消息的实时性、可靠性、顺序性等。

### Informer 机制架构设计

![Informer运行原理](../image/informer运行原理.jpg)

1. Reflector

Reflector 用于监控（Watch）指定的 Kubernetes 资源，当监控的资源发送变化时，触发相应的变更事件（例如 Added,Updated,Deleted），并将其资源对象存放到本地缓存 DeltaFIFO 中。

2. DeltaFIFO 

DeltaFIFO 可以分开理解，FIFO 是一个先进先出的队列，它拥有队列操作的基本方法，例如Add,Update,Delete,List,Pop,Close等，而 Delta 是一个资源对象存储，它可以保存资源对象的操作类型，例如Added,Updated,Deleted,Sync操作类型等。

3. Indexer

Indexer 是 client-go 用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO 中将消费出来的资源对象存储至 Indexer。Indexer 与 Etcd 集群中的数据完全保持一致。client-go 可以很方便地从本地存储中读取相应的资源对象数据，而无须每次从远程 Etcd 集群中读取，以减轻 Kubernetes API Server 和 Etcd 集群的压力。

Informers Example 代码示例如下：

```
func main() {
    config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
    if err != nil {
        panic(err)
    }

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err)
    }

    stopCh := make(chan struct{})
    defer close(stopCh)

    sharedInformers := informers.NewSharedInformerFactory(clientset, time.Minute)
    informer := sharedInformers.Core().V1().Pods().Informer()

    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            mObj := obj.(v1.Object)
            log.Printf("New Pod Added to Store: %s", mObj.GetName())
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oObj := oldObj.(v1.Object)
            nObj := newObj.(v1.Object)
            log.Printf("%s Pod Updated to %s", oObj.GetName(), nObj.GetName())
        },
        DeleteFunc: func(obj interface{}) {
            mObj := obj.(v1.Object)
            log.Printf("Pod Deleted from Store: %s", mObj.GetName())
        },
    })

    informer.Run(stopCh)
}
```

首先通过 kubernetes.NewForConfig 创建 clientset 对象，Informer 需要通过 ClientSet 与 Kubernetes API Server 进行交互。另外创建 stopCh 对象，该对象用于在程序进程退出之前通知 Informer 提前退出，因为 Informer 是一个持久运行的 goroutine。

* informers.NewSharedInformerFactory 函数实例化了 SharedInformer 对象，它接受两个参数，第一个参数 clientset 是用于与 Kubernetes API Server 交互的客户端，第二个参数 time.Minute 用于设置多久进行一次重新同步（resync），resync 会周期性地执行 List 操作，将所有的资源存放在 Informer Store 中，如果该参数为 0，则禁用 resync 功能。

* 通过 sharedInformers.Core().V1().Pods().Informer() 可以得到具体的 Pod 资源的 Informer 对象。

* 通过 informer.AddEventHandler 函数可以为 Pod 资源添加资源事件回调方法。

> 正常情况下， Kubernetes 的其他组件在使用 Informer 机制时触发资源事件回调方法，将资源对象推送到 WorkQueue 或其他队列中。

通过 Informer 机制可以很容易地监控我们所关心的资源事件，并通知 client-gp，告知 Kubernetes 资源事件变更了并且需要进行相应的处理。

1. 资源 Informer 

每一个 Kubernetes 资源上都实现了 Inforemer 机制，每一个 Informer 上都会实现 Informer 和 Lister 方法，例如 PodInformer,代码示例如下：

```
type PodInformer interface {
    Informer() cache.SharedIndexInformer
    Lister()  v1.PodLister
}
```

调用不同资源的 Informer，代码示例如下：

```
podInformer := sharedInformer.Core().V1().Pods().Informer()
nodeInformer := sharedInformer.Node().V1beta1().RuntimeClasses().Informer()
```

定义不同资源的 Informe，允许监控不同资源的资源事件。

2. Shared Informer 共享机制

Informer 也被称为 Shared Informer，它是可以共享使用的。若同一资源的 Informer 被实例化了多次，每个 Informer 使用一个 Reflector，那么会运行过多相同的 ListAndWatch，太多重复的序列化和反序列化操作会导致 Kubernetes API Server 负载过重。

Shared Informer 可以使同一资源 Informer 共享一个 Reflector，这样可以节约很多资源。通过 map 数据结构实现共享的 Informer 机制。Shared Informer 定义了一个 map 数据结构，用于存放所有 Informer 的字段，代码示例如下：

```
type sharedInformerFactory struct {
    ...
    informers map[reflect.Type]cache.SharedIndexInformer
}

func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer{
    ...
    informerType := reflect.TypeOf(obj)
    informer, exists := f.informers[informerType]
    if exists {
        return informer
    }
    ...
    f.informers[informerType] = informer

    return informer
}
```

* informers 字段中存储了资源类型和对应于 SharedIndexInformer 的映射关系。
* informerFor 函数添加了不同资源的 Informer，在添加过程中如果已经存在同类型的资源 Informer，则返回当前 Informer，不再继续添加。
* 最后通过 Shared Informer 的 Start 方法使 f.informers 中的每个 informer 通过 goroutine 持久运行。

### Reflector

Informer 可以监控 Kubernetes 内置资源，也可以是 CRD 自定义资源，其中最核心的功能是 Refector。Reflector 用于监控（Watch）指定的 Kubernetes 资源，当监控的资源发送变化时，触发相应的变更事件（例如 Added,Updated,Deleted），并将其资源对象存放到本地缓存 DeltaFIFO 中。

* 通过 NewReflector 实例化 Reflector 对象，实例化过程中传入 ListerWatcher 数据接口对象，它拥有 List 和 Watch 方法，用于获取监控资源列表。（只要实现了 Listere 和 Watch 方法的对象都可以称为 ListerWatcher)

* ListAndWatcher 函数实现可分为两部分：第一部分获取资源列表数据，第二部分监控资源对象。

1. 获取资源列表数据

ListAndWatch List 在程序第一次运行时获取该资源下所有的对象数据并将其存储至 DeltaFIFO 中。

（1）r.listerWatcher.List 用于获取资源下所有对象的数据，ResourceVersion 为 0，则表示获取所有 Pod 的资源数据，非 0 则表示根据资源版本号继续获取，可以使本地缓存中的数据和 Etcd 中的数据保持一致。
（2）listMetaInterface.GetResourceVersion 用于获取资源版本号，Kubernetes 中所有的资源都拥有该字段，每次修改当前资源对象时，Kubernetes API Server 都会更改 ResourceVersion，使得 client-go 执行 Watch 操作时可以根据 ResourceVersion 来确定当前资源对象是否发生变化。
（3）meta.ExtractList 用于将资源数据转换成资源对象列表，将 runtime.Object 对象转换成 []runtime.Object 对象。
（4）r.syncWith 用于将资源对象列表中的资源对象和资源版本号存储至 DeltaFIFO 中，并会替换已存在的对象。
（5）r.setLastSyncResourceVersion 用于设置最新的资源版本号。

代码示例如下：

```
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
    ...
    var resourceVersion string
    options := metav1.ListOptions{ResourceVersion: "0"}

    if err := func() error {
        ...
        list, err = r.listerWatcher.List(options)
        ...
        listMetaInterface, err := meta.ListAccessor(list)
        ...
        resourceVersion = listMetaInterface.GetResourceVersion()
        items, err := meta.ExtractList(list)
        ...
        if err := r.syncWith(items, resourceVersion); err != nil {
            ...
        }
        r.setLastSyncResourceVersion(resourceVersion)
        ...
    }(); err != nil {
        return err
    }
    ...
}
```

r.listerWatcher.List  函数实际调用了 Pod Informer 下的 ListFunc 函数，它通过 ClientSet 客户端与 Kubernetes API Server 交互并获取 Pod 资源列表数据，代码示例如下：

```
ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
    if tweakListOptions != nil {
        tweakListOptions(&options)
    }
    return client.CoreV1().Pods(namespace).List(options)
},
```

2. 监控资源对象

Watch 操作通过 HTTP 协议与 Kubernetes API Server 建立长连接，接收 Kubernetes API Server 发来的资源变更事件。Watch 操作的实现机制使用 HTTP 协议的分块传输编码。

ListAndWatch Watch 代码示例如下：

```
for {
    ...
    timeoutSeconds := int64(minWatchTimeout.Seconds()*(rand.Float64() + 1.0))
    options = metav1.ListOptions{
        ResourceVersion: resourceVersion,
        TimeoutSeconds: &timeoutSeconds,
    }

    w, err := r.listerWatcher.Watch(options)
    ...
    if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
        ...
        return nil
    }
}
```

r.listerWatcher.Watch 函数实际调用了 Pod Informer 下的 WatchFunc 函数，它通过 ClientSet 客户端与 Kubernetes API Server 建立长连接，监控指定资源的变更事件，代码示例如下：

```
WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
    if tweakListOptions != nil {
        tweakListOptions(&options)
    }
    return client.CoreV1().Pods(namespace).Watch(options)
},
```

r.watchHandler 用于处理资源的变更事件。当触发 Added,Updated,Deleted 事件时，将对应的资源对象更新到本地缓存 DeltaFIFO 中，并更新 ResourceVersion 资源版本号。r.watchHandler 代码示例如下：

```
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <- struct{}) error {
    ...
    for {
        select {
            ...
            case event, ok := <- w.ResultChan():
                ...
                switch event.Type {
                    case watch.Added:
                        err := r.store.Add(event.Object)
                        ...
                    case watch.Modified:
                        err := r.store.Updated(event.Object)
                        ...
                    case watch.Deleted:
                        err := r.store.Delete(event.Object)
                        ...
                    default:
                        ...
                }
                *resourceVersion = newResourceVersion
                r.setLastSyncResourceVersion(newResourceVersion)
                ...
        }
    }
    ...
}
```

### DeltaFIFO

DeltaFIFO 可以分开理解，FIFO 是一个先进先出的队列，它拥有队列操作的基本方法，例如Add,Update,Delete,List,Pop,Close等，而 Delta 是一个资源对象存储，它可以保存资源对象的操作类型，例如 Added,Updated,Deleted,Sync操作类型等。

DeltaFIFO 结构代码示例如下：

```
type DeltaFIFO struct {
    ...
    items map[string]Deltas
    // 该字段通过 map 数据结构的方式存储，value 存储的是对象的 Deltas 数组。
    queue []string
    // 该字段存储资源对象的 key，该 key 通过 Keyof 函数计算得到。
    ...
}
type Deltas []Delta
```

DeltaFIFO 与其他队列最大的不同之处是，它会保留所有关于资源对象的操作类型，队列中会存在拥有不同操作类型的同一资源对象，消费者在处理该资源对象时能够了解该资源对象所发生的事情。

DeltaFIFO 本质上是一个先进先出的队列，有数据的生产者和消费者，其中生产者是 Reflect 调用的 Add 方法，消费者是 Controller 调用的 Pop 方法。

1. 生产者方法

DeltaFIFO 队列中的资源对象在 Added,Updated,Deleted 事件中都调用了 queueActionLocked 函数，它是 DeltaFIFO 实现的关键。代码示例如下：

```
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
    id, err := f.KeyOf(obj)
    ...
    if actionType == Sync && f.willObjectBeDeletedLocked(id) {
        return nil
    }

    newDeltas := append(f.items[id], Delta(actionType, obj))
    newDeltas = dedupDelats(newDeltas)

    if len(newDeltas) > 0 {
        if _, exists := f.items[id]; !exists {
            f.queue = append(f.queue, id)
        }
        f.items[id] = newDeltas
        f.cond.Broadcast()
    } else {
        delete(f.items, id)
    }
    return nil
}
```

queueActionLocked 代码执行流程如下：

（1）通过 f.KeyOf 函数计算出资源对象的 key.
（2）如果操作类型为 Sync，则标识该数据来源于 Indexer (本地存储)，如果 Indexer 中的资源对象已经被删除，则直接返回。
（3）将 actionType 和资源对象构造成 Delta，添加到 items 中，并通过 dedupDelta 函数进行去重操作。
（4）更新构造后的 Delta 并通过 cond.Broadcast 通过所有消费者解除阻塞。

2. 消费者方法

Pop 方法从 DeltaFIFO 的头部取出最早进入队列中的资源对象数据，Pop 方法必须传入 process 函数，用于接收并处理对象的回调方法，代码示例如下：

```
func (f *DeltaFIF) Pop (process PopProcessFunc) (interface{}, error) {
    ...
    for {
        for len(f.queue) == 0 {
            ...
            f.cond.Wait()
            // 如果队列中没有数据时，通过 f.cond.wait 阻塞等待数据。
        }
        id := f.queue[0]
        f.queue = f.queue[1:]
        ...
        item, ok := f.items[id]
        ...
        delete(f.items, id)
        err := process(item)
        if e, ok := err.(ErrRequeue); ok {
            f.addIfNoPresent(id, item)
            err = e.Err
        }
        // 如果 process 回调函数处理出错，则将该对象重新存入队列。
        return item, err
    }
}
```

Controller 的 processLoop 方法负责将从 DeltaFIFO 队列中取出数据传递给 process 回调函数，process 回调函数代码示例如下：

```
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
    ...
    switch d.Type {
        case Sync, Added, Updated:
            ...
            if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
                if err := s.indexer.Update(d.Object); err != nil {
                    return err
                }
                s.processor.distribute(updateNotification{oldObj; old, newObj: d.Object}, isSync)
            } else {
                if err := s.indexer.Add(d.Object); err != nil {
                    return err
                }
                s.processor.distribute(addNotification{newObj: d.Object}, isSync)
            }
        case Deleted:
            if err := s.indexer.Delete(d.Object); err != nil {
                return err
            }
            s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
    }
    ...
}
```

* HandleDeltas 函数，当资源对象的操作类型为 Added、Updated、Deleted 时,将资源对象存储至 Indexer，并通过 distribute 函数将资源对象分发至 SharedInformer。
> 通过 informer.AddEventHandler 函数添加了对资源事件进行处理的函数，distribute 函数则将资源对象分发到该事件处理函数中。

3. Resync 机制

Resync 机制会将 Indexer 本地存储中的资源对象同步到 DeltaFIFO 中，并将这些资源对象设置为 Sync 的操作类型。Resync -> syncKeyLocked 代码示例如下：

```
func (f *DeltaFIFO) syncKeyLocked (key string) error {
    obj, exists, err := f.knownObjects.GetByKey(key)
    ...
    id, err := f.KeyOf(obj)
    ...
    if err := f.queueActionLocked(Sync, obj); err != nil {
        return fmt.Errorf("couldn't queue object: %v", err)
    }
    return nil
}
```

f.knownObjects 是 Indexer 本地存储对象，通过该对象可以获取 client-go 目前存储的所有资源对象。

### Indexer

Indexer 是 client-go 用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO 中将消费出来的资源对象存储至 Indexer。Indexer 中的数据与 Etcd 集群中的数据保持完全一致。client-go 可以很方便地从本地存储中读取相应的资源对象数据，而无须每次从远程 Etcd 集群中读取，以减轻 Kubernetes API Server 和 Etcd 集群的压力

* ThreadSafeMap 是实现并发安全的存储，作为存储，它拥有存储相关的增、删、改、查操作方法。

* Indexer 在 ThreadSafeMap 的基础上进行了封装，它继承了与 ThreadSafeMap 相关的操作方法并实现了 Indexer Func 等功能，例如 Index, IndexKeys, GetInderers 等方法，这些方法为 ThreadSafeMap 提供了索引功能。

1. ThreadfSafeMap 并发安全存储

ThreadfSafeMap 是内存中的存储，其中的数据并不会写入本地磁盘中，每次的增、删、改、查操作都会加锁，以保证数据的一致性。ThreadfSafeMap 将资源对象数据存储于一个 map 数据结构中，ThreadfSafeMap 结构代码示例如下：

```
type threadSafeMap struct {
    items map[string]interface{}
    ...
}
```

items 字段中存储的是资源对象数据，其中 items 的 key 通过 keyFunc 函数计算得到(默认使用 MetaNamespaceKeyFunc 函数，该函数根据资源对象计算出<namespqce>/<name>格式的 key)，vlue 用于存储资源对象。

2. Indexer 索引器

在每次增、删、改 ThreadSafeMap 数据时，都会通过 updateIndices 或 deleteFromIndices 函数变更 Indexer。Indexer 被设计为可以自定义索引函数，Indexer Example 代码示例如下：

```
func UsersIndexFunc(obj interface{}) ([]string, error) {
    pod := obj.(*v1.Pod)
    usersString := pod.Annotations["users"]

    return strings.Split(usersString, ","), nil
}

func main() {
    index := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{"byUser": UsersIndexFunc})

    pod1 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "one", Annotations: map[string]string{"users": "ernie,bert"}}}
    pod2 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "two", Annotations: map[string]string{"users": "bert,oscar"}}}
    pod3 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "tre", Annotations: map[string]string{"users": "ernie,elmo"}}}

    index.Add(pod1)
    index.Add(pod2)
    index.Add(pod3)

    erniePods, err := index.ByIndex("byUser", "ernie")
    if err != nil {
        panic(err)
    }

    for _, erniePod := range erniePods {
        fmt.Println(erniePod.(*v1.Pod).Name)
    }
}
```

* 首先定义一个索引器函数 UserIndexFunc，在该函数中，我们定义查询所有 Pod 资源下 Annotations 字段的 key 为 users 的 Pod。

* cache.NewIndexer 函数实例化了 Indexer 对象，该函数接收两个参数：第一个参数是 KeyFunc，使用 MetaNamespaceKeyFunc 函数，该函数根据资源对象计算出<namespqce>/<name>格式的 key，第二个参数是 cashe.Indexers，用于定义索引器，其中 key 为索引器的名称，value 为索引器。

* 最后通过 index.Add 函数添加资源对象，并通过 索引去查询。

Indexers、IndexFunc、Indices、Index 数据结构如下：

```
// 存储索引器，key 为索引器名称，value 为索引器的实现函数。
type Indexers map[string]IndexFunc

// 索引器函数，定义为接收一个资源对象，返回检索结果列表
type IndexFunc func(obj interface{}) ([]string, error)

// 存储缓存器，key 为缓存器名称，value 为缓存数据
type Indices map[string]Index

// 存储缓存数据，其结构为 K/V
type Index map[string]sets.String
```

3. Indexer 索引器核心实现

index.ByIndex 函数通过执行索引器函数得到索引结果，代码示例如下：

```
func (c *threadSafeMap) ByIndex (indexName, indexKey string) ([]interface{}, error) {
    ...
    indexFunc := c.indexers[indexName]
    ...
    index := c.indices[indexName]

    set := index[indexKey]
    list := make([]interface{}, 0, set.Len())
    for _, key := range set.List() {
        list = append(list, c.items[key])
    }

    return list, nil
}
```

ByIndex 接收两个参数：索引器名称和需要检索的 key，首先从 c.indices 中查找指定的缓存器函数，然后根据需要检索的 indexKey 从缓存数据中查到并返回数据。

> Index 中的缓存数据为 Set 集合数据结构，Set 本质与 Slice 相同，但 Set 中不存在相同元素，由于 Go 语言标准库没有提供 Set 数据结构，Go 语言中的 map 结构类型是不能存在相同 key 的，所以 K8S 将 map 结构类型的 key 作为 Set 数据结构，实现 Set 去重特性。