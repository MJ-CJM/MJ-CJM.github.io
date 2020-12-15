---
title: client-go3.0(workqueue)
date: 2020-12-07 22:08:20
tags: [Kubernetes, client-go]
categories: Kubernetes源码解析
---

WorkQueue 称为工作队列，Kubernetes 的 WorkQueue 队列与普通 FIFO 队列相比，它的主要功能在于标记和去重，并支持如下特性：

* 有序：按照添加顺序处理元素。
* 去重：相同元素在同一时间不会被重复处理。
* 并发性：多生产者和多消费者
* 标记机制：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队。
* 通知机制：ShutDown 方法通过信号量通知队列不再接收新的元素，并通知 metric goroutine 退出。
* 延迟：支持延迟队列，延迟一段时间后再将元素存入队列。
* 限速：支持限速队列，元素存入队列时进行速率限制。（限制一个元素被重新排队的次数）
* Metric: 支持 metric 监控指标，可用于 Prometheus 监控。

WorkQueue 支持三种队列，并提供了 3 种接口：

1. Interface: FIFO 队列接口，先进先出队列，并支持去重机制。
2. DelayingInterface: 延迟队列接口，基于 Interface 接口封装，延迟一段时间后再将元素存入队列。
3. RateLimitingInterface: 限速队列接口，基于 DelayingInterface 接口封装，支持元素存入队列时进行速率限制。

### FIFO 队列

FIFO 队列支持最基本的队列方法，WorkQueue 中的限速及延迟队列都基于 Interface 接口实现，其提供如下方法：

```
type Interface interface {
    Add(item interface{})
    // 给队列添加元素，可以是任意类型的
    Len() int
    // 返回当前队列的长度
    Get() (item interface{}, shutdown bool)
    // 获取队列头部的一个元素
    Done(item interface{})
    // 标记队列中该元素已被处理
    ShutDown()
    // 关闭队列
    ShuttingDown() bool
    // 查询队列是否正在关闭
}
```

FIFO 队列数据结构如下：

```
type Type struct {
    queue []t
    // queue是实际存储元素的地方
    dirty set
    // 除了保证能去重，还能保证一个元素只被处理一次
    processing set
    // 用于标记一个元素是否正在被处理
    cond *sync.Cond
    shuttingDown bool
    metrics queueMetrics
    unfinishedWorkUpdatedPeriod time.Duration
    clock clock.Clock
}
```

* 一旦元素完成被 Done 方法标记，则将该元素添加到 queue 字段的末尾。

* dirty 和 processing 字段都是用 Hash Map 数据结构实现的，所以不需要考虑无序，只保证去重即可。

### 延迟队列

延迟队列，基于 FIFO 队列接口封装，在原有功能上增加了 AddAfter 方法，其原理是延迟一段时间后再将元素插入 FIFO 队列。延迟队列数据结构如下：

```
type DelayingInterface interface {
    Interface
    AddAfter(item interface{}, duration time.Duration)
    // 插入一个元素，duration 为延迟时间参数，小于或等于0，会直接将元素插入 FIFO 队列中。
}

type delayingType struct {
    Interface
    clock clock.Clock
    stopCh chan struct{}
    heartbeat clock.Ticker
    waitingForAddCh chan *waitFor
    // 其默认初始大小为1000，通过 AddAfter 方法插入元素时，是非阻塞状态，当插入元素大于或等于 1000 时，延迟队列才会处于阻塞状态。
    metrics retryMetrics
    deprecatedMetrics retryMetrics
}
```

将元素放入 waitingForAddCh 字段中，通过 waitingLoop 函数消费元素数据，当元素的延迟时间不大于当前时间，说明还需要延迟将元素插入 FIFO 队列的时间，此时将该元素放入优先队列中，当元素的延迟时间大于当前时间时，则将该元素插入 FIFO 队列中，另外，还会遍历优先队列中的元素，按照上述逻辑验证时间。

### 限速队列

限速队列，基于延迟队列和 FIFO 队列接口封装（RateLimitingInterface），重点在于它提供的 4 种限速算法接口(RateLimiter)，其原理是，限速队列利用延迟队列的特性，延迟某个元素的插入时间，达到限速目的，RateLimiter 数据结构如下：

```
type RateLimiter interface {
    When(item interface{}) time.Duration
    // 获取指定元素应该等待的时间
    Forget(item interface{})
    // 释放指定元素，清空该元素的排队数
    NumRequeues(item interface{}) int
    // 获取指定元素的排队数
}
```

> 限速周期：一个限速周期是指从执行 AddRateLimited 方法到执行完 Forget 方法之间的时间，如果该元素被 Forget 方法处理完，则清空排队数。

#### 令牌桶算法

令牌桶算法内部实现了一个存放 token 的 “桶”，初始时“桶”是空的，token 会以固定速率往“桶”里填充，直到将其填满为止，多余的 token 会被丢弃。每个元素会从令牌桶得到一个 token，只有得到 token 的元素才允许通过，而没得到 token 的元素处于等待状态。令牌桶算法通过发放 token 来达到限速目的。

WorkQueue 在默认的情况下会实例化令牌桶，代码示例如下：

```
rate.NewLimiter(rate.Limit(10), 100)
```

其中传入 r 和 b 两个参数，r 表示每秒往“桶”里填充的 token 数量，b 参数表示令牌桶的大小，那么前 b 个元素会被立刻处理，而后面元素的延迟时间分别为 item100/100ms, item101/200ms,item102/300ms,item103/400ms。

#### 排队指数算法


#### 计数器算法


#### 混合模式






