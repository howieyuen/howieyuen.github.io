---
author: Yuan Hao
date: 2020-10-25
title: GC Controller 源码分析
tag: [garbage collector, controller]
---

# 1. 序言 

垃圾回收相关，可参考这里[垃圾回收]({{< relref "/docs/kubernetes/sig-api-machinery/garbage-collector.md" >}})

# 2. 源码解析

GarbageCollectorController 负责回收集群中的资源对象，要做到这一点，首先得监控所有资源。gc controller 会监听集群中所有可删除资源的事件，这些事件会放到一个队列中，然后启动多个 worker 协程处理。对于删除事件，则根据删除策略删除对象；其他事件，更新对象之间的依赖关系。

## 2.1 startGarbageCollectorController()

首先来看 gc controller 的入口方法，也就是 kube-controller-manager 是如何启动它的。它的主要逻辑：
1. 判断是否启用 gc controller，默认是true
2. 初始化 clientset，使用 discoveryClient 获取集群中所有资源
3. 注册不考虑 gc 的资源，默认为空
4. 调用 `garbagecollector.NewGarbageCollector()` 方法 初始化 gc controller 对象
5. 调用 `garbageCollector.Run()` 启动 gc controller，workers 默认是 20
6. 调用 `garbageCollector.Sync()` 监听集群中的资源，当出现新的资源时，同步到 minitors 中
7. 调用 `garbagecollector.NewDebugHandler()` 注册 debug 接口，用来提供集群内所有对象的关联关系；

`cmd/kube-controller-manager/app/core.go:538`
```go
func startGarbageCollectorController(ctx ControllerContext) (http.Handler, bool, error) {
	// 1.判断是否启用 gc controller，默认是true
	if !ctx.ComponentConfig.GarbageCollectorController.EnableGarbageCollector {
		return nil, false, nil
	}
	// 2.初始化 clientset
	gcClientset := ctx.ClientBuilder.ClientOrDie("generic-garbage-collector")

	config := ctx.ClientBuilder.ConfigOrDie("generic-garbage-collector")
	metadataClient, err := metadata.NewForConfig(config)
	if err != nil {
		return nil, true, err
	}
	// 3.注册不考虑 gc 的资源，默认为空
	ignoredResources := make(map[schema.GroupResource]struct{})
	for _, r := range ctx.ComponentConfig.GarbageCollectorController.GCIgnoredResources {
		ignoredResources[schema.GroupResource{Group: r.Group, Resource: r.Resource}] = struct{}{}
	}
	// 4.初始化 gc controller 对象
	garbageCollector, err := garbagecollector.NewGarbageCollector(
		metadataClient,
		ctx.RESTMapper,
		ignoredResources,
		ctx.ObjectOrMetadataInformerFactory,
		ctx.InformersStarted,
	)
	if err != nil {
		return nil, true, fmt.Errorf("failed to start the generic garbage collector: %v", err)
	}

	// 5.启动 gc，workers 默认是20
	workers := int(ctx.ComponentConfig.GarbageCollectorController.ConcurrentGCSyncs)
	go garbageCollector.Run(workers, ctx.Stop)

	// Periodically refresh the RESTMapper with new discovery information and sync
	// the garbage collector.
	// 6.监听集群中的资源，周期更新
	go garbageCollector.Sync(gcClientset.Discovery(), 30*time.Second, ctx.Stop)

	// 7.注册 debug 接口
	return garbagecollector.NewDebugHandler(garbageCollector), true, nil
}
```

在 `startGarbageCollectorController()` 中主要调用了4个方法：
- `garbagecollector.NewGarbageCollector()`
- `garbageCollector.Run()`
- `garbageCollector.Sync()`
- `garbagecollector.NewDebugHandler()`
其中 `garbagecollector.NewGarbageCollector()` 只是初始化 GarbageCollector 和 GraphBuilder 对象，核心逻辑都在 Run() 和 Sync() 中，下面分别来看这几个方法分别做了那些事。

### 2.1.1 garbageCollector.Run()

`garbageCollector.Run()` 方法主要作用是启动生产者和消费者。生产者就是 monitors，监听集群中的资源对象，将产生的新事件分别放入 `attemptToDelete` 和 `attemptToOrphan` 两个队列中。消费者就是处理这 2 个队列中的事件，要么删除对象，要么更新对象依赖关系。该方法的核心在于 `gc.dependencyGraphBuilder.Run()` 启动生产者和 for 循环启动消费者。

`pkg/controller/garbagecollector/garbagecollector.go:122`
```go
func (gc *GarbageCollector) Run(workers int, stopCh <-chan struct{}) {
	...

	// 1. 启动所有 monitors 即 informers，监听集群资源
	// 并启动一个协程，处理 graphChanges 中事件，
	// 然后经过处理，分别放入 GraphBuilder 的 attemptToDelete 和 attemptToOrphan 两个队列中
	go gc.dependencyGraphBuilder.Run(stopCh)
	
	// 2. 等待 informers 的 cache 同步完成
	if !cache.WaitForNamedCacheSync("garbage collector", stopCh, gc.dependencyGraphBuilder.IsSynced) {
		return
	}

	klog.Infof("Garbage collector: all resource monitors have synced. Proceeding to collect garbage")

	// gc workers
	for i := 0; i < workers; i++ {
		// 3. 启动多个协程，处理 attemptToDelete 队列中的事件
		go wait.Until(gc.runAttemptToDeleteWorker, 1*time.Second, stopCh)
		// 4. 启动多个协程，处理 attemptToOrphan 队列中的事件
		go wait.Until(gc.runAttemptToOrphanWorker, 1*time.Second, stopCh)
	}

	<-stopCh
}
```

**GraphBuilder**
GraphBuilder 在整个垃圾收集的过程中，起到了承上启下的作用。首先看下它的结构：

`pkg/controller/garbagecollector/graph_builder.go:73`
```go
type GraphBuilder struct {
	restMapper meta.RESTMapper

	// monitor 就是 informer，一个monitor list/watch 一种资源
	monitors    monitors
	monitorLock sync.RWMutex
	
	// 当 kube-controller-manager 中所有的 controllers 都启动后，就会被 close 掉
	informersStarted <-chan struct{}

	// stopCh 是接收 shutdown 信号
	stopCh <-chan struct{}

	// 当调用 GraphBuilder 的 run 方法时，running 会被设置为 true
	running bool

	metadataClient metadata.Interface
	
	// monitors 监听到的事件会放在 graphChanges 中
	graphChanges workqueue.RateLimitingInterface
	
	// uidToNode 维护所有对象的依赖关系
	uidToNode *concurrentUIDToNode
	
	// GarbageCollector 作为消费者要处理 attemptToDelete 和 attemptToOrphan 两个队列中的事件
	attemptToDelete workqueue.RateLimitingInterface
	attemptToOrphan workqueue.RateLimitingInterface
	
	// absentOwnerCache 存放已知不存在的对象
	absentOwnerCache *UIDCache
	sharedInformers  controller.InformerFactory
	// 不需要被 gc 的资源，默认是空
	ignoredResources map[schema.GroupResource]struct{}
}
```
其中 `uidToNode` 字段作为维护对象之间依赖关系，比如创建一个 Deployment 时，会创建 ReplicaSet，ReplicaSet 才会创建 Pod。那么 Pod 的 owner 是 ReplicaSet，ReplicaSet 的 owner 是 Deployment。`uidToNode` 字段的结构定义如下：

`pkg/controller/garbagecollector/graph.go:162`
```go
type concurrentUIDToNode struct {
	uidToNodeLock sync.RWMutex
	uidToNode     map[types.UID]*node
}

type node struct {
	identity objectReference

	dependentsLock sync.RWMutex

	// dependents 保存的是 owner 是自己的对象集合
	dependents map[*node]struct{}
	// this is set by processGraphChanges() if the object has non-nil DeletionTimestamp
	// and has the FinalizerDeleteDependents.
	deletingDependents     bool
	deletingDependentsLock sync.RWMutex
	// this records if the object's deletionTimestamp is non-nil.
	beingDeleted     bool
	beingDeletedLock sync.RWMutex
	
	// 如果 virtual 为 true，表示这个对象不是通过 informer 观察到的，是虚拟构建的
	virtual     bool
	virtualLock sync.RWMutex
	
	// owners 保存的是自己的 owner
	owners []metav1.OwnerReference
}
```
`GraphBuilder` 主要有三个功能：
1. 监控集群中所有的可删除资源；
2. 基于 informers 中的资源在 `uidToNode` 数据结构中维护着所有对象的依赖关系；
3. 处理 graphChanges 中的事件并放到 `attemptToDelete` 和 `attemptToOrphan` 两个队列中；

**gc.dependencyGraphBuilder.Run()**
继续回到 `gc.dependencyGraphBuilder.Run()` 方法，它的功能上文已经提到，就是启动生产者。代码如下：

`pkg/controller/garbagecollector/graph_builder.go:290`

```go
func (gb *GraphBuilder) Run(stopCh <-chan struct{}) {
	klog.Infof("GraphBuilder running")
	defer klog.Infof("GraphBuilder stopping")

	// 1. 设置 stop channel
	gb.monitorLock.Lock()
	gb.stopCh = stopCh
	gb.running = true
	gb.monitorLock.Unlock()

	// 2. 启动 monitor，除非 stop channel 收到信号
	gb.startMonitors()
	// 调用 gb.runProcessGraphChanges，分类事件，放入2个队列中
	// 此处为死循环，除非收到 stopCh 信号，否则下面的代码不会被执行到
	wait.Until(gb.runProcessGraphChanges, 1*time.Second, stopCh)

	// 代码走到这里，说明 stopCh 收到了信号，停止所有 monitor
	gb.monitorLock.Lock()
	defer gb.monitorLock.Unlock()
	monitors := gb.monitors
	stopped := 0
	for _, monitor := range monitors {
		if monitor.stopCh != nil {
			stopped++
			close(monitor.stopCh)
		}
	}

	// reset monitors so that the graph builder can be safely re-run/synced.
	gb.monitors = nil
	klog.Infof("stopped %d of %d monitors", stopped, len(monitors))
}
```

继续看 `gb.runProcessGraphChanges()` 方法，这里是生产者的核心逻辑。总结一下：
1. 从 `graphChanges` 队列中取出一个 item 即 event；
2. 获取 event 的 accessor，accessor 是一个 object 的 meta.Interface，里面包含访问 object meta 中所有字段的方法；
3. 通过 accessor 获取 UID 判断 `uidToNode` 中是否存在该 object；

根据对象是否存在以及事件类型，分成三种情况
- 若 `uidToNode` 中不存在该 node 且该事件是 addEvent 或 updateEvent

则为该 object 创建对应的 node，并调用 `gb.insertNode()` 将该 node 加到 `uidToNode` 中，然后将该 node 添加到其 owner 的 dependents 中，执行完 `gb.insertNode()` 中的操作后再调用 `gb.processTransitions()` 方法判断该对象是否处于删除状态，若处于删除状态会判断该对象是以 orphan 模式删除还是以 foreground 模式删除，若以 orphan 模式删除，则将该 node 加入到 `attemptToOrphan` 队列中，若以 foreground 模式删除则将该对象以及其所有 dependents 都加入到 `attemptToDelete` 队列中；

- 若 `uidToNode` 中存在该 node 且该事件是 addEvent 或 updateEvent 

此时可能是一个 update 操作，调用 `referencesDiffs()` 方法检查该对象的 OwnerReferences 字段是否有变化，若有变化
1. 调用 `gb.addUnblockedOwnersToDeleteQueue()` 将被删除以及更新的 owner 对应的 node 加入到 `attemptToDelete` 中，因为此时该 node 中已被删除或更新的 owner 可能处于删除状态且阻塞在该 node 处，此时有三种方式避免该 node 的 owner 处于删除阻塞状态，一是等待该 node 被删除，二是将该 node 自身对应 owner 的 `OwnerReferences `字段删除，三是将该 node 的 `OwnerReferences` 字段中对应 owner 的 `BlockOwnerDeletion` 设置为 false；
2. 更新该 node 的 owners 列表；
3. 若有新增的 owner，将该 node 加入到新 owner 的 dependents 中；
4. 若有被删除的 owner，将该 node 从已删除 owner 的 dependents 中删除；以上操作完成后，检查该 node 是否处于删除状态并进行标记，最后调用 `gb.processTransitions()` 方法检查该 node 是否要被删除；

举个例子，若以 foreground 模式删除 deployment 时，deployment 的 dependents 列表中有对应的 rs，那么 deployment 的删除会阻塞住等待其依赖 rs 的删除，此时 rs 有三种方法不阻塞 deployment 的删除操作，一是 rs 对象被删除，二是删除 rs 对象 OwnerReferences 字段中对应的 deployment，三是将 rs 对象 OwnerReferences 字段中对应的 deployment 配置 BlockOwnerDeletion 设置为 false，文末会有示例演示该操作。

- 若该事件为 deleteEvent
首先从 `uidToNode` 中删除该对象，然后从该 node 所有 owners 的 dependents 中删除该对象，将该 node 所有的 dependents 加入到 `attemptToDelete` 队列中，最后检查该 node 的所有 owners，若有处于删除状态的 owner，此时该 owner 可能处于删除阻塞状态正在等待该 node 的删除，将该 owner 加入到 `attemptToDelete` 中；

`pkg/controller/garbagecollector/graph_builder.go:530`
```go
func (gb *GraphBuilder) runProcessGraphChanges() {
	for gb.processGraphChanges() {
	}
}

func (gb *GraphBuilder) processGraphChanges() bool {
	// 1. 从 graphChanges 取出一个 event
	item, quit := gb.graphChanges.Get()
	if quit {
		return false
	}
	defer gb.graphChanges.Done(item)
	event, ok := item.(*event)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect a *event, got %v", item))
		return true
	}
	obj := event.obj
	accessor, err := meta.Accessor(obj)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("cannot access obj: %v", err))
		return true
	}
	klog.V(5).Infof("GraphBuilder process object: %s/%s, namespace %s, name %s, uid %s, event type %v", event.gvk.GroupVersion().String(), event.gvk.Kind, accessor.GetNamespace(), accessor.GetName(), string(accessor.GetUID()), event.eventType)
	// 2. 若存在 node 对象，从 uidToNode 中取出该 event 的 node 对象
	existingNode, found := gb.uidToNode.Read(accessor.GetUID())
	if found {
		existingNode.markObserved()
	}
	switch {
	// 3.1 若 event 为 add 或 update 类型且对应的 node 对象不存在时，表示新建对象
	case (event.eventType == addEvent || event.eventType == updateEvent) && !found:
		// 3.1.1 为 node 创建 event 对象
		newNode := &node{
			identity: objectReference{
				OwnerReference: metav1.OwnerReference{
					APIVersion: event.gvk.GroupVersion().String(),
					Kind:       event.gvk.Kind,
					UID:        accessor.GetUID(),
					Name:       accessor.GetName(),
				},
				Namespace: accessor.GetNamespace(),
			},
			dependents:         make(map[*node]struct{}),
			owners:             accessor.GetOwnerReferences(),
			deletingDependents: beingDeleted(accessor) && hasDeleteDependentsFinalizer(accessor),
			beingDeleted:       beingDeleted(accessor),
		}
		// 3.1.2 在 uidToNode 中添加该 node 对象
		gb.insertNode(newNode)
		// 3.1.3 检查对象的删除策略，并放到对应的队列中
		gb.processTransitions(event.oldObj, accessor, newNode)
	// 3.2 若 event 为 add 或 update 类型且对应的 node 对象存在时，表示更新对象
	case (event.eventType == addEvent || event.eventType == updateEvent) && found:
		// 3.2.1 检查当前对象和老对象的owner的不同
		added, removed, changed := referencesDiffs(existingNode.owners, accessor.GetOwnerReferences())
		if len(added) != 0 || len(removed) != 0 || len(changed) != 0 {
			// 3.2.2 检查依赖变更是否解除了处于等待依赖删除的owner的阻塞状态
			gb.addUnblockedOwnersToDeleteQueue(removed, changed)
			// 3.2.3 更新 node 本身
			existingNode.owners = accessor.GetOwnerReferences()
			// 3.2.4 添加 node 到它的 owner 依赖中
			gb.addDependentToOwners(existingNode, added)
			// 3.2.5 删除老的 owners 中的依赖
			gb.removeDependentFromOwners(existingNode, removed)
		}

		if beingDeleted(accessor) {
			existingNode.markBeingDeleted()
		}
		// 3.2.6 检查对象的删除策略，并放到对应的队列中
		gb.processTransitions(event.oldObj, accessor, existingNode)
	// 3.3 若为 delete event
	case event.eventType == deleteEvent:
		if !found {
			klog.V(5).Infof("%v doesn't exist in the graph, this shouldn't happen", accessor.GetUID())
			return true
		}
		// 3.3.1 从 uidToNode 中删除该 node
		gb.removeNode(existingNode)
		existingNode.dependentsLock.RLock()
		defer existingNode.dependentsLock.RUnlock()
		if len(existingNode.dependents) > 0 {
			gb.absentOwnerCache.Add(accessor.GetUID())
		}
		// 3.3.2 删除该 node 的 dependents
		for dep := range existingNode.dependents {
			gb.attemptToDelete.Add(dep)
		}
		// 3.3.2 删除该 node 处于删除阻塞状态的 owner
		for _, owner := range existingNode.owners {
			ownerNode, found := gb.uidToNode.Read(owner.UID)
			if !found || !ownerNode.isDeletingDependents() {
				continue
			}
			gb.attemptToDelete.Add(ownerNode)
		}
	}
	return true
}
```

### 2.1.2 gc.runAttemptToDeleteWorker()

`gc.runAttemptToDeleteWorker()` 方法就是将 `attemptToDelete` 队列中的对象取出，并删除，如果删除失败则重进队列重试。

`pkg/controller/garbagecollector/garbagecollector.go:285`
```go
func (gc *GarbageCollector) runAttemptToDeleteWorker() {
	for gc.attemptToDeleteWorker() {
	}
}

func (gc *GarbageCollector) attemptToDeleteWorker() bool {
	// 取出对象
    item, quit := gc.attemptToDelete.Get()
	gc.workerLock.RLock()
	defer gc.workerLock.RUnlock()
	if quit {
		return false
	}
	defer gc.attemptToDelete.Done(item)
	n, ok := item.(*node)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect *node, got %#v", item))
		return true
	}
    // 实际执行删除的方法 gc.attemptToDeleteItem()
	err := gc.attemptToDeleteItem(n)
	if err != nil {
		if _, ok := err.(*restMappingError); ok {
			klog.V(5).Infof("error syncing item %s: %v", n, err)
		} else {
			utilruntime.HandleError(fmt.Errorf("error syncing item %s: %v", n, err))
		}
        // 删除失败则重新进入队列等待重试
		gc.attemptToDelete.AddRateLimited(item)
	} else if !n.isObserved() {
		klog.V(5).Infof("item %s hasn't been observed via informer yet", n.identity)
		// 如果 node 不是 informer 发现的，则也要删除
		gc.attemptToDelete.AddRateLimited(item)
	}
	return true
}
```
`gc.runAttemptToDeleteWorker()` 中调用了 `gc.attemptToDeleteItem()` 执行实际的删除操作。下面继续来看 `gc.attemptToDeleteItem()` 的实现细节：

1. 判断 node 是否处于删除状态；
2. 从 apiserver 获取该 node 最新的状态，该 node 可能为 virtual node，若为 virtual node 则从 apiserver 中获取不到该 node 的对象，此时会将该 node 重新加入到 graphChanges 队列中，再次处理该 node 时会将其从 uidToNode 中删除；
3. 判断该 node 最新状态的 uid 是否等于本地缓存中的 uid，若不匹配说明该 node 已更新过此时将其设置为 virtual node 并重新加入到 graphChanges 队列中，再次处理该 node 时会将其从 uidToNode 中删除；
4. 通过 node 的 deletingDependents 字段判断该 node 当前是否处于删除 dependents 的状态，若该 node 处于删除 dependents 的状态则调用 processDeletingDependentsItem 方法检查 node 的 blockingDependents 是否被完全删除，若 blockingDependents 已完全被删除则删除该 node 对应的 finalizer，若 blockingDependents 还未删除完，将未删除的 blockingDependents 加入到 attemptToDelete 中；上文中在 GraphBuilder 处理 graphChanges 中的事件时，若发现 node 处于删除状态，会将 node 的 dependents 加入到 attemptToDelete 中并标记 node 的 deletingDependents 为 true；
5. 调用 gc.classifyReferences 将 node 的 ownerReferences 分类为 solid, dangling, waitingForDependentsDeletion 三类：dangling(owner 不存在)、waitingForDependentsDeletion(owner 存在，owner 处于删除状态且正在等待其 dependents 被删除)、solid(至少有一个 owner 存在且不处于删除状态)；对以上分类进行不同的处理：
    1. 第一种情况是若 solid不为 0 即当前 node 至少存在一个 owner，该对象还不能被回收，此时需要将 dangling 和 waitingForDependentsDeletion 列表中的 owner 从 node 的 ownerReferences 删除，即已经被删除或等待删除的引用从对象中删掉；
    2. 第二种情况是该 node 的 owner 处于 waitingForDependentsDeletion 状态并且 node 的 dependents 未被完全删除，该 node 需要等待删除完所有的 dependents 后才能被删除；
    3. 第三种情况就是该 node 已经没有任何 dependents 了，此时按照 node 中声明的删除策略调用 apiserver 的接口删除即可；

`pkg/controller/garbagecollector/garbagecollector.go:409`
```go
func (gc *GarbageCollector) attemptToDeleteItem(item *node) error {
	klog.V(2).InfoS("Processing object", "object", klog.KRef(item.identity.Namespace, item.identity.Name),
		"objectUID", item.identity.UID, "kind", item.identity.Kind)

	// 1. 判断 node 是否处于删除状态
	if item.isBeingDeleted() && !item.isDeletingDependents() {
		klog.V(5).Infof("processing item %s returned at once, because its DeletionTimestamp is non-nil", item.identity)
		return nil
	}
	
	// 2. 从 apiserver 获取该 node 最新的状态
	latest, err := gc.getObject(item.identity)
	switch {
	case errors.IsNotFound(err):
		klog.V(5).Infof("item %v not found, generating a virtual delete event", item.identity)
		gc.dependencyGraphBuilder.enqueueVirtualDeleteEvent(item.identity)
		item.markObserved()
		return nil
	case err != nil:
		return err
	}
	
	// 3. 判断该 node 最新状态的 uid 是否等于本地缓存中的 uid
	if latest.GetUID() != item.identity.UID {
		klog.V(5).Infof("UID doesn't match, item %v not found, generating a virtual delete event", item.identity)
		gc.dependencyGraphBuilder.enqueueVirtualDeleteEvent(item.identity)
		item.markObserved()
		return nil
	}

	// 4. 判断该 node 当前是否处于删除 dependents 状态中
	if item.isDeletingDependents() {
		return gc.processDeletingDependentsItem(item)
	}
	
	// 5. 检查 node 是否还存在 ownerReferences
	ownerReferences := latest.GetOwnerReferences()
	if len(ownerReferences) == 0 {
		klog.V(2).Infof("object %s's doesn't have an owner, continue on next item", item.identity)
		return nil
	}
	
	// 6. 对 ownerReferences 进行分类
	solid, dangling, waitingForDependentsDeletion, err := gc.classifyReferences(item, ownerReferences)
	if err != nil {
		return err
	}
	klog.V(5).Infof("classify references of %s.\nsolid: %#v\ndangling: %#v\nwaitingForDependentsDeletion: %#v\n", item.identity, solid, dangling, waitingForDependentsDeletion)

	switch {
	// 7.1 存在不处于删除状态的 owner
	case len(solid) != 0:
		klog.V(2).Infof("object %#v has at least one existing owner: %#v, will not garbage collect", item.identity, solid)
		if len(dangling) == 0 && len(waitingForDependentsDeletion) == 0 {
			return nil
		}
		klog.V(2).Infof("remove dangling references %#v and waiting references %#v for object %s", dangling, waitingForDependentsDeletion, item.identity)
		ownerUIDs := append(ownerRefsToUIDs(dangling), ownerRefsToUIDs(waitingForDependentsDeletion)...)
		patch := deleteOwnerRefStrategicMergePatch(item.identity.UID, ownerUIDs...)
		_, err = gc.patch(item, patch, func(n *node) ([]byte, error) {
			return gc.deleteOwnerRefJSONMergePatch(n, ownerUIDs...)
		})
		return err
	// 7.2 node 的 owner 处于 waitingForDependentsDeletion 状态并且 node 的 dependents 未被完全删除
	case len(waitingForDependentsDeletion) != 0 && item.dependentsLength() != 0:
		deps := item.getDependents()
		// 删除 dependents
		for _, dep := range deps {
			if dep.isDeletingDependents() {
				klog.V(2).Infof("processing object %s, some of its owners and its dependent [%s] have FinalizerDeletingDependents, to prevent potential cycle, its ownerReferences are going to be modified to be non-blocking, then the object is going to be deleted with Foreground", item.identity, dep.identity)
				patch, err := item.unblockOwnerReferencesStrategicMergePatch()
				if err != nil {
					return err
				}
				if _, err := gc.patch(item, patch, gc.unblockOwnerReferencesJSONMergePatch); err != nil {
					return err
				}
				break
			}
		}
		klog.V(2).Infof("at least one owner of object %s has FinalizerDeletingDependents, and the object itself has dependents, so it is going to be deleted in Foreground", item.identity)
		// 以 Foreground 模式删除 node 对象
		policy := metav1.DeletePropagationForeground
		return gc.deleteObject(item.identity, &policy)
	// 7.3 该 node 已经没有任何依赖了，按照 node 中声明的删除策略调用 apiserver 的接口删除
	default:
		var policy metav1.DeletionPropagation
		switch {
		case hasOrphanFinalizer(latest):
			policy = metav1.DeletePropagationOrphan
		case hasDeleteDependentsFinalizer(latest):
			policy = metav1.DeletePropagationForeground
		default:
			policy = metav1.DeletePropagationBackground
		}
		klog.V(2).InfoS("Deleting object", "object", klog.KRef(item.identity.Namespace, item.identity.Name),
			"objectUID", item.identity.UID, "kind", item.identity.Kind, "propagationPolicy", policy)
		return gc.deleteObject(item.identity, &policy)
	}
}
```

### 2.1.3 gc.runAttemptToOrphanWorker()

`gc.runAttemptToOrphanWorker()` 是处理以 Orphan 模式删除的 node，主要逻辑为：

1. 调用 `gc.orphanDependents()` 删除 owner 所有 dependents OwnerReferences 中的 owner 字段；
2. 调用 `gc.removeFinalizer()` 删除 owner 的 orphan Finalizer；
3. 以上两步中若有失败的会进行重试；

`pkg/controller/garbagecollector/garbagecollector.go:602`
```go   
func (gc *GarbageCollector) attemptToOrphanWorker() bool {
	item, quit := gc.attemptToOrphan.Get()
	gc.workerLock.RLock()
	defer gc.workerLock.RUnlock()
	if quit {
		return false
	}
	defer gc.attemptToOrphan.Done(item)
	owner, ok := item.(*node)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect *node, got %#v", item))
		return true
	}
	// we don't need to lock each element, because they never get updated
	owner.dependentsLock.RLock()
	dependents := make([]*node, 0, len(owner.dependents))
	for dependent := range owner.dependents {
		dependents = append(dependents, dependent)
	}
	owner.dependentsLock.RUnlock()

	err := gc.orphanDependents(owner.identity, dependents)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("orphanDependents for %s failed with %v", owner.identity, err))
		gc.attemptToOrphan.AddRateLimited(item)
		return true
	}
	// update the owner, remove "orphaningFinalizer" from its finalizers list
	err = gc.removeFinalizer(owner, metav1.FinalizerOrphanDependents)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("removeOrphanFinalizer for %s failed with %v", owner.identity, err))
		gc.attemptToOrphan.AddRateLimited(item)
	}
	return true
}
```

### 2.1.4 小结

上面的业务逻辑不算复杂，但是方法嵌套有点多，整理一下方法的调用链：

{{< mermaid >}}
graph LR

p1(garbageCollector.Run) --> p11(gc.dependencyGraphBuilder.Run)

p11 --> p111(gb.startMonitors)

p11 --> p112(gb.runProcessGraphChanges)

p112 --> p1121(gb.processGraphChanges)

p1121 --> p11211(gb.processTransitions)

p1 --> p12(gc.runAttemptToDeleteWorker)

p12 --> p121(gc.attemptToDeleteWorker)

p121 --> p1211(gc.attemptToDeleteItem)

p1 --> p13(gc.runAttemptToOrphanWorker)

p13 --> p131(gc.orphanDependents)

p13 --> p132(gc.removeFinalizer)
{{< /mermaid >}}

## 2.2 garbageCollector.Sync()

`garbageCollector.Sync()` 方法主要是周期性地查询集群中的所有资源，过滤出 deletableResource，然后对比已经监控的 deletableResource 是否一致，如果不一致，则更新 GraphBuilder 的 monitors，并重启 monitors 监控新拿到的 deletableResource。主要逻辑：
1. 通过调用 `GetDeletableResources()` 获取集群内所有的 deletableResources 作为 newResources，deletableResources 指支持 “delete”, “list”, “watch” 三种操作的 resource，包括自定义资源
2. 检查 oldResources, newResources 是否一致，不一致则需要同步；
3. 调用 `gc.resyncMonitors()` 同步 newResources，在 `gc.resyncMonitors()` 中会重新调用 GraphBuilder 的 `syncMonitors()` 和 `startMonitors()` 两个方法完成 monitors 的刷新；
4. 等待 newResources informer 中的 cache 同步完成；
5. 将 newResources 作为 oldResources，继续进行下一轮的同步； 

`pkg/controller/garbagecollector/garbagecollector.go:168`

```go
func (gc *GarbageCollector) Sync(discoveryClient discovery.ServerResourcesInterface, period time.Duration, stopCh <-chan struct{}) {
	oldResources := make(map[schema.GroupVersionResource]struct{})
	wait.Until(func() {
		// 1. 获取集群内所有的 deletableResources 作为 newResources
		newResources := GetDeletableResources(discoveryClient)

		// 正常情况下是不会走到这里，除非发生内部错误
		if len(newResources) == 0 {
			klog.V(2).Infof("no resources reported by discovery, skipping garbage collector sync")
			return
		}

		// 2. 判断集群中的资源是否有变化
		if reflect.DeepEqual(oldResources, newResources) {
			klog.V(5).Infof("no resource updates from discovery, skipping garbage collector sync")
			return
		}

		// 加锁保证在 informer 同步完成后处理 event
		gc.workerLock.Lock()
		defer gc.workerLock.Unlock()

		// 3. 开始更新 GraphBuilder 中的 monitors
		attempt := 0
		wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
			attempt++

			// 尝试次数大于 1，先判断 deletableResource 是否改变
			if attempt > 1 {
				newResources = GetDeletableResources(discoveryClient)
				if len(newResources) == 0 {
					klog.V(2).Infof("no resources reported by discovery (attempt %d)", attempt)
					return false, nil
				}
			}

			klog.V(2).Infof("syncing garbage collector with updated resources from discovery (attempt %d): %s", attempt, printDiff(oldResources, newResources))

			gc.restMapper.Reset()
			klog.V(4).Infof("reset restmapper")

			// 4. 调用 gc.resyncMonitors 同步 newResources
			if err := gc.resyncMonitors(newResources); err != nil {
				utilruntime.HandleError(fmt.Errorf("failed to sync resource monitors (attempt %d): %v", attempt, err))
				return false, nil
			}
			klog.V(4).Infof("resynced monitors")

			// 5. 等待所有 monitors 的 cache 同步完成
			if !cache.WaitForNamedCacheSync("garbage collector", waitForStopOrTimeout(stopCh, period), gc.dependencyGraphBuilder.IsSynced) {
				utilruntime.HandleError(fmt.Errorf("timed out waiting for dependency graph builder sync during GC sync (attempt %d)", attempt))
				return false, nil
			}

			return true, nil
		}, stopCh)

		// 6. 更新 oldResources
		oldResources = newResources
		klog.V(2).Infof("synced garbage collector")
	}, period, stopCh)
}
```

方法主要调用了 `GetDeletableResources()` 和 `gc.resyncMonitors()` 两个方法。前者获取集群中可删除资源，后者更新 monitors。

### 2.2.1 GetDeletableResources()

`GetDeletableResources()` 中首先通过调用 `discoveryClient.ServerPreferredResources()` 方法获取集群内所有的 resource 信息，然后通过调用 `discovery.FilteredBy()` 过滤出支持 “delete”, “list”, “watch” 三种方法的 resource 作为 deletableResources。

`pkg/controller/garbagecollector/garbagecollector.go:658`
```go
func GetDeletableResources(discoveryClient discovery.ServerResourcesInterface) map[schema.GroupVersionResource]struct{} {
	// 获取集群内所有的 resource 信息
	preferredResources, err := discoveryClient.ServerPreferredResources()
	...
	if preferredResources == nil {
		return map[schema.GroupVersionResource]struct{}{}
	}

	// 过滤出 deletableResources
	deletableResources := discovery.FilteredBy(discovery.SupportsAllVerbs{Verbs: []string{"delete", "list", "watch"}}, preferredResources)
	deletableGroupVersionResources := map[schema.GroupVersionResource]struct{}{}
	for _, rl := range deletableResources {
		gv, err := schema.ParseGroupVersion(rl.GroupVersion)
		if err != nil {
			klog.Warningf("ignoring invalid discovered resource %q: %v", rl.GroupVersion, err)
			continue
		}
		for i := range rl.APIResources {
			deletableGroupVersionResources[schema.GroupVersionResource{Group: gv.Group, Version: gv.Version, Resource: rl.APIResources[i].Name}] = struct{}{}
		}
	}

	return deletableGroupVersionResources
}
```

### 2.2.2 gc.resyncMonitors()

`gc.resyncMonitors()` 的功能主要是更新 GraphBuilder 的 monitors 并重新启动 monitors 监控所有的 deletableResources，GraphBuilder 的 `startMonitors()` 方法在前面的流程中已经分析过，此处不再详细说明。`syncMonitors()` 只不过是拿最新的 deletableResources，把老的 monitors 字段值更新，该删的删，该加的加而已。

`pkg/controller/garbagecollector/garbagecollector.go:113`
```go
func (gc *GarbageCollector) resyncMonitors(deletableResources map[schema.GroupVersionResource]struct{}) error {
	if err := gc.dependencyGraphBuilder.syncMonitors(deletableResources); err != nil {
		return err
	}
	gc.dependencyGraphBuilder.startMonitors()
	return nil
}
```

## 2.3 garbagecollector.NewDebugHandler()

`garbagecollector.NewDebugHandler()` 主要功能是对外提供一个接口供用户查询当前集群中所有资源的依赖关系，依赖关系可以以图表的形式展示。

```go
func NewDebugHandler(controller *GarbageCollector) http.Handler {
	return &debugHTTPHandler{controller: controller}
}

func (h *debugHTTPHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if req.URL.Path != "/graph" {
		http.Error(w, "", http.StatusNotFound)
		return
	}

	var graph graph.Directed
	if uidStrings := req.URL.Query()["uid"]; len(uidStrings) > 0 {
		uids := []types.UID{}
		for _, uidString := range uidStrings {
			uids = append(uids, types.UID(uidString))
		}
		graph = h.controller.dependencyGraphBuilder.uidToNode.ToGonumGraphForObj(uids...)

	} else {
		graph = h.controller.dependencyGraphBuilder.uidToNode.ToGonumGraph()
	}

	data, err := dot.Marshal(graph, "full", "", "  ")
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "text/vnd.graphviz")
	w.Header().Set("X-Content-Type-Options", "nosniff")
	w.Write(data)
	w.WriteHeader(http.StatusOK)
}
```

使用该服务的方法如下：
```sh
curl http://192.168.199.100:10252/debug/controllers/garbagecollector/graph  > tmp.dot

curl http://192.168.199.100:10252/debug/controllers/garbagecollector/graph?uid=f9555d53-2b5f-4702-9717-54a313ed4fe8 > tmp.dot

// 生成 svg 文件
$ dot -Tsvg -o graph.svg tmp.dot

// 然后在浏览器中打开 svg 文件
```

依赖关系如下图所示（点击图片查看更多）：

[![tmp.jpg](/kubernetes/sig-apimachinery/gc/tmp.jpg)](/kubernetes/sig-apimachinery/gc/graph.svg)

## 2.4 总结

`GarbageCollectorController` 是一种典型的生产者消费者模型，所有 deletableResources 的 informer 都是生产者，每种资源的 informer 监听到变化后都会将对应的事件 push 到 `graphChanges` 中，`graphChanges` 是 `GraphBuilder` 对象中的一个数据结构，`GraphBuilder` 会启动另外的 goroutine 对 graphChanges 中的事件进行分类并放在其 `attemptToDelete` 和 `attemptToOrphan` 两个队列中，garbageCollector 会启动多个 goroutine 对 `attemptToDelete` 和 `attemptToOrphan` 两个队列中的事件进行处理，处理的结果就是回收一些需要被删除的对象。最后，再用一个流程图总结一下 `GarbageCollectorController` 的主要流程:

{{< mermaid >}}
graph LR
a[mirrors] -->|produce| b(graphChanges)
b --> c(processGraphChanges)
c --> d(attemptToDelete)
c --> e(attemptToOrphan)
d -->|consume| f[AttemptToDeleteWorker]
e -->|consume| g[AttemptToOrphanWorker]
{{< /mermaid >}}


# 3. 参考资料

- [垃圾收集](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection)
- [garbage collector controller 源码分析](https://cloud.tencent.com/developer/article/1562130)
- [Kubernetes API 资源对象的删除和 GarbageCollector Controller](http://yangxikun.github.io/kubernetes/2020/03/17/kubernetes-delete-obj-and-garbage-collector-controller.html)