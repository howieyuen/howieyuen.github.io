---
author: Yuan Hao
date: 2020-09-24
title: k8s watch 关键设计
tag: [watch, client-go, kube-apiserver, etcd]
weight: 1
---

> 注：本文转自[图解kubernetes中基于etcd的watch关键设计](https://www.kubernetes.org.cn/6889.html)

本文介绍了 kubernetes 针对 etcd 的 watch 场景，k8s 在性能优化上面的一些设计，
逐个介绍缓存、定时器、序列化缓存、bookmark 机制、forget 机制、
针对数据的索引与 ringbuffer 等组件的场景以及解决的问题，
希望能帮助到那些对 apiserver 中的 watch 机制实现感兴趣的朋友。

# 1. 事件驱动与控制器

![image1](/kubernetes/sig-apimachinery/watch/image1.png)

k8s 中并没有将业务的具体处理逻辑耦合在 rest 接口中，rest 接口只负责数据的存储，
通过控制器模式，分离数据存储与业务逻辑的耦合，保证 apiserver 业务逻辑的简洁。


![image2](/kubernetes/sig-apimachinery/watch/image2.png)

控制器通过 watch 接口来感知对应的资源的数据变更，从而根据资源对象中的期望状态与当前状态之间的差异，
来决策业务逻辑的控制，watch 本质上做的事情其实就是将感知到的事件发生给关注该事件的控制器。

# 2. Watch 的核心机制

这里我们先介绍基于 etcd 实现的基础的 watch 模块。

## 2.1 事件类型与 etcd

![image3](/kubernetes/sig-apimachinery/watch/image3.png)

一个数据变更本质上无非就是三种类型：新增、更新和删除，
其中新增和删除都比较容易因为都可以通过当前数据获取，而更新则可能需要获取之前的数据，
这里其实就是借助了 etcd 中 revision 和 mvcc 机制来实现，这样就可以获取到之前的状态和更新后的状态，
并且获取后续的通知。

## 2.2 事件管道

![image4](/kubernetes/sig-apimachinery/watch/image4.png)

事件管道则是负责事件的传递，在 watch 的实现中通过两级管道来实现消息的分发，
首先通过 watch etcd 中的 key 获取感兴趣的事件，并进行数据的解析，
完成从bytes到内部事件的转换并且发送到输入管道(incomingEventChan)中，
然后后台会有线程负责输入管道中获取数据，并进行解析发送到输出管道(resultChan)中，
后续会从该管道来进行事件的读取发送给对应的客户端。

## 2.3 事件缓冲区

事件缓冲区是指的如果对应的事件处理程序与当前事件发生的速率不匹配的时候，
则需要一定的 buffer 来暂存因为速率不匹配的事件，
在 go 里面大家通常使用一个有缓冲的 channel 构建。

![image5](/kubernetes/sig-apimachinery/watch/image5.png)

到这里基本上就实现了一个基本可用的 watch 服务，通过 etcd 的 watch 接口监听数据，
然后启动独立 goroutine 来进行事件的消费，并且发送到事件管道供其他接口调用。

# 3. Cacher

kubernetes 中所有的数据和系统都基于 etcd 来实现，如何减轻访问压力呢，
答案就是缓存，watch 也是这样，本节我们来看看如何实现 watch 缓存机制的实现，
这里的 cacher 是针对 watch 的。

## 3.1 Reflector

![image6](/kubernetes/sig-apimachinery/watch/image6.png)

Reflector 是 client-go 中的一个组件，其通过listwatch接口获取数据存储在自己内部的 store 中，
cacher中通过该组件对 etcd 进行 watch 操作，避免为每个组件都创建一个 etcd 的 watcher。

## 3.2 watchCache

![image7](/kubernetes/sig-apimachinery/watch/image7.png)

wacthCache 负责存储 watch 到的事件，并且将 watch 的事件建立对应的本地索引缓存，
同时在构建 watchCache 还负责将事件的传递，
其将 watch 到的事件通过 eventHandler 来传递给上层的 Cacher 组件。

## 3.3 cacheWatcher

![image8](/kubernetes/sig-apimachinery/watch/image8.png)

cacheWatcher 顾名思义其是就是针对 cache 的一个 watcher(watch.Interface)实现，
前端的 watchServer 负责从 ResultChan 里面获取事件进行转发。

## 3.4 Cacher

![image9](/kubernetes/sig-apimachinery/watch/image9.png)

Cacher 基于 etcd 的 store 结合上面的 watchCache 和 Reflector 共同构建带缓存的 REST store，
针对普通的增删改功能其直接转发给 etcd 的 store 来进行底层的操作，而对于 watch 操作则进行拦截，
构建并返回 cacheWatcher 组件。

# 4. Cacher的优化

看完基础组件的实现，接着我们看下针对watch这个场景k8s中还做了那些优化，学习针对类似场景的优化方案。

## 4.1 序列化缓存

![image10](/kubernetes/sig-apimachinery/watch/image10.png)

如果我们有多个 watcher 都 wacth 同一个事件，在最终的时候我们都需要进行序列化，
cacher 中在分发的时候，如果发现超过指定数量的watcher， 则会在进行 dispatch 的时候，
为其构建构建一个缓存函数，针对多个 watcher 只会进行一次的序列化。

## 4.2 nonblocking

![image11](/kubernetes/sig-apimachinery/watch/image11.png)

在上面我们提到过事件缓冲区，但是如果某个 watcher 消费过慢依然会影响事件的分发，
为此 cacher 中通过是否阻塞(是否可以直接将数据写入到管道中)来将 watcher 分为两类，
针对不能立即投递事件的 watcher， 则会在后续进行重试。

## 4.3 TimeBudget

针对阻塞的 watcher 在进行重试的时候，会通过 dispatchTimeoutBudget 构建一个定时器来进行超时控制，
那什么叫 Budget 呢，其实如果在这段时间内，如果重试立马就成功，则本次剩余的时间，
在下一次进行定时的时候，则可以使用之前剩余的余额，但是后台也还有个线程，用于周期性重置。

## 4.4 forget机制

![image12](/kubernetes/sig-apimachinery/watch/image12.png)

针对上面的 TimeBudget 如果在给定的时间内依旧无法进行重试成功，
则就会通过 forget 来删除对应的 watcher， 
由此针对消费特别缓慢的 watcher 则可以通过后续的重试来重新建立 watch，
从而减小对a piserver 的 watch 压力。

## 4.5 bookmark 机制

![image13](/kubernetes/sig-apimachinery/watch/image13.png)

bookmark机制是大阿里提供的一种优化方案，其核心是为了避免单个某个资源一直没有对应的事件，
此时对应的 informer 的 revision 会落后集群很大，
bookmark 通过构建一种 BookMark 类型的事件来进行 revision 的传递，
从而让 informer 在重启后不至于落后特别多。

## 4.6 watchCache 中的 ringbuffer

![image14](/kubernetes/sig-apimachinery/watch/image14.png)

watchCache 中通过 store 来构建了对应的索引缓存，但是在 listwatch 操作的时候，
则通常需要获取某个 revision 后的所有数据，
针对这类数据 watchCache 中则构建了一个 ringbuffer 来进行历史数据的缓存。

# 5. 设计总结

![image15](/kubernetes/sig-apimachinery/watch/image15.png)

本文介绍了 kubernetes 针对 etcd 的 watch 场景，k8s 在性能优化上面的一些设计，
逐个介绍缓存、定时器、序列化缓存、bookmark 机制、forget 机制、
针对数据的索引与 ringbuffer 等组件的场景以及解决的问题，
希望能帮助到那些对 apiserver 中的 watch 机制实现感兴趣的朋友。