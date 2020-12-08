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



