---
author: Yuan Hao
date: 2020-09-24
title: Key Design of ETCD Watch
tag: [watch, client-go, kube-apiserver, etcd]
weight: 1
---

<!-- ## 图解kubernetes中基于etcd的watch关键设计 -->

本文介绍了kubernetes针对etcd的watch场景，k8s在性能优化上面的一些设计，逐个介绍缓存、定时器、序列化缓存、bookmark机制、forget机制、针对数据的索引与ringbuffer等组件的场景以及解决的问题，希望能帮助到那些对apiserver中的watch机制实现感兴趣的朋友。

### 1. 事件驱动与控制器

![image1](/kubernetes/sig-apimachinery/watch/image1.png)

k8s中并没有将业务的具体处理逻辑耦合在rest接口中，rest接口只负责数据的存储，通过控制器模式，分离数据存储与业务逻辑的耦合，保证apiserver业务逻辑的简洁。


![image2](/kubernetes/sig-apimachinery/watch/image2.png)

控制器通过watch接口来感知对应的资源的数据变更，从而根据资源对象中的期望状态与当前状态之间的差异，来决策业务逻辑的控制，watch本质上做的事情其实就是将感知到的事件发生给关注该事件的控制器。

### 2. Watch的核心机制

这里我们先介绍基于etcd实现的基础的watch模块。

#### 2.1 事件类型与etcd

![image3](/kubernetes/sig-apimachinery/watch/image3.png)

一个数据变更本质上无非就是三种类型：新增、更新和删除，其中新增和删除都比较容易因为都可以通过当前数据获取，而更新则可能需要获取之前的数据，这里其实就是借助了etcd中revision和mvcc机制来实现，这样就可以获取到之前的状态和更新后的状态，并且获取后续的通知。

#### 2.2 事件管道

![image4](/kubernetes/sig-apimachinery/watch/image4.png)

事件管道则是负责事件的传递，在watch的实现中通过两级管道来实现消息的分发，首先通过watch etcd中的key获取感兴趣的事件，并进行数据的解析，完成从bytes到内部事件的转换并且发送到输入管道(incomingEventChan)中，然后后台会有线程负责输入管道中获取数据，并进行解析发送到输出管道(resultChan)中，后续会从该管道来进行事件的读取发送给对应的客户端。

#### 2.3 事件缓冲区

事件缓冲区是指的如果对应的事件处理程序与当前事件发生的速率不匹配的时候，则需要一定的buffer来暂存因为速率不匹配的事件， 在go里面大家通常使用一个有缓冲的channel构建。

![image5](/kubernetes/sig-apimachinery/watch/image5.png)

到这里基本上就实现了一个基本可用的watch服务，通过etcd的watch接口监听数据，然后启动独立goroutine来进行事件的消费，并且发送到事件管道供其他接口调用。

### 3. Cacher

kubernetes中所有的数据和系统都基于etcd来实现，如何减轻访问压力呢，答案就是缓存，watch也是这样，本节我们来看看如何实现watch缓存机制的实现，这里的cacher是针对

#### 3.1 Reflector

![image6](/kubernetes/sig-apimachinery/watch/image6.png)

Reflector是client-go中的一个组件，其通过listwatch接口获取数据存储在自己内部的store中，cacher中通过该组件对etcd进行watch操作，避免为每个组件都创建一个etcd的watcher。

#### 3.2 watchCache

![image7](/kubernetes/sig-apimachinery/watch/image7.png)

wacthCache负责存储watch到的事件，并且将watch的事件建立对应的本地索引缓存，同时在构建watchCache还负责将事件的传递，其将watch到的事件通过eventHandler来传递给上层的Cacher组件。

#### 3.3 cacheWatcher

![image8](/kubernetes/sig-apimachinery/watch/image8.png)

cacheWatcher顾名思义其是就是针对cache的一个watcher(watch.Interface)实现， 前端的watchServer负责从ResultChan里面获取事件进行转发。

#### 3.4 Cacher

![image9](/kubernetes/sig-apimachinery/watch/image9.png)

Cacher基于etcd的store结合上面的watchCache和Reflector共同构建带缓存的REST store， 针对普通的增删改功能其直接转发给etcd的store来进行底层的操作，而对于watch操作则进行拦截，构建并返回cacheWatcher组件。

### 4. Cacher的优化

看完基础组件的实现，接着我们看下针对watch这个场景k8s中还做了那些优化，学习针对类似场景的优化方案。

#### 4.1 序列化缓存

![image10](/kubernetes/sig-apimachinery/watch/image10.png)

如果我们有多个watcher都wacth同一个事件，在最终的时候我们都需要进行序列化，cacher中在分发的时候，如果发现超过指定数量的watcher， 则会在进行dispatch的时候，为其构建构建一个缓存函数，针对多个watcher只会进行一次的序列化。

#### 4.2 nonblocking

![image11](/kubernetes/sig-apimachinery/watch/image11.png)

在上面我们提到过事件缓冲区，但是如果某个watcher消费过慢依然会影响事件的分发，为此cacher中通过是否阻塞(是否可以直接将数据写入到管道中)来将watcher分为两类，针对不能立即投递事件的watcher， 则会在后续进行重试。

#### 4.3 TimeBudget

针对阻塞的watcher在进行重试的时候，会通过dispatchTimeoutBudget构建一个定时器来进行超时控制， 那什么叫Budget呢，其实如果在这段时间内，如果重试立马就成功，则本次剩余的时间，在下一次进行定时的时候，则可以使用之前剩余的余额，但是后台也还有个线程，用于周期性重置。

#### 4.4 forget机制

![image12](/kubernetes/sig-apimachinery/watch/image12.png)

针对上面的TimeBudget如果在给定的时间内依旧无法进行重试成功，则就会通过forget来删除对应的watcher， 由此针对消费特别缓慢的watcher则可以通过后续的重试来重新建立watch，从而减小对apiserver的watch压力。

#### 4.5 bookmark机制

![image13](/kubernetes/sig-apimachinery/watch/image13.png)

bookmark机制是大阿里提供的一种优化方案，其核心是为了避免单个某个资源一直没有对应的事件，此时对应的informer的revision会落后集群很大，bookmark通过构建一种BookMark类型的事件来进行revision的传递，从而让informer在重启后不至于落后特别多。

#### 4.6 watchCache中的ringbuffer

![image14](/kubernetes/sig-apimachinery/watch/image14.png)

watchCache中通过store来构建了对应的索引缓存，但是在listwatch操作的时候，则通常需要获取某个revision后的所有数据，针对这类数据watchCache中则构建了一个ringbuffer来进行历史数据的缓存。

### 5. 设计总结

![image15](/kubernetes/sig-apimachinery/watch/image15.png)

本文介绍了kubernetes针对etcd的watch场景，k8s在性能优化上面的一些设计，逐个介绍缓存、定时器、序列化缓存、bookmark机制、forget机制、针对数据的索引与ringbuffer等组件的场景以及解决的问题，希望能帮助到那些对apiserver中的watch机制实现感兴趣的朋友。
