---
author: Yuan Hao
date: 2020-10-12
title: Statefulset Controller 源码分析
tag: [statefulset, controller]
weight: 1
---

# 1. StatefulSet 简介

Statefulset 是为了解决有状态服务的问题，而产生的一种资源类型（Deployment 和 ReplicaSet 是解决无状态服务而设计的）。

这里可能有人说，MySQL 是有状态服务吧，但我使用的是 Deploment 资源类型，MySQL 的数据通过 PV 的方式存储在第三方文件系统中，也能解决 MySQL 数据存储问题。

是的，如果你的 MySQL 是单节点，使用 Deployment 类型确实可以解决数据存储问题。但是如果你的有状态服务是集群，且每个节点分片存储的情况下，Deployment 则不适用这种场景，因为 Deployment 不会保证 Pod 的有序性，集群通常需要主节点先启动，从节点在加入集群，Statefulset 则可以保证，其次 Deployment 资源的 Pod 内的 PVC 是共享存储的，而 Statefulset 下的 Pod 内 PVC 是不共享存储的，每个 Pod 拥有自己的独立存储空间，正好满足了分片的需求，实现分片的需求的前提是 Statefulset 可以保证 Pod 重新调度后还是能访问到相同的持久化数据。

适用 Statefulset 常用的服务有 Elasticsearch 集群，Mogodb集群，Redis 集群等等。

## 1.1 特点

- 稳定、唯一的网络标识符

    如: Redis 集群，在 Redis 集群中，它是通过槽位来存储数据的，假如：第一个节点是 0~1000，第二个节点是 1001~2000，第三个节点 2001~3000……，这就使得 Redis 集群中每个节点要通过 ID 来标识自己，如：第二个节点宕机了，重建后它必须还叫第二个节点，或者说第二个节点叫 R2，它必须还叫 R2，这样在获取 1001~2000 槽位的数据时，才能找到数据，否则Redis集群将无法找到这段数据。

- 稳定、持久的存储
  
  可实现持久存储，新增或减少 Pod，存储不会随之发生变化。

- 有序的、平滑的部署、扩容
    
    如 MySQL 集群，要先启动主节点， 若从节点没有要求，则可一起启动，若从节点有启动顺序要求，可先启动第一个从节点，接着第二从节点等；这个过程就是有顺序，平滑安全的启动。

- 有序的、自动的缩容和删除
    
    我们先终止从节点，若从节点是有启动顺序的，那么关闭时，也要按照逆序终止，即启动时是从 S1~S4 以此启动，则关闭时，则是先关闭 S4，然后是 S3，依次关闭，最后在关闭主节点。

- 有序的、自动的滚动更新

    MySQL 在更新时，应该先更新从节点，全部的从节点都更新完了，最后在更新主节点，因为新版本一般可兼容老版本，但是一定要注意，若新版本不兼容老版本就很很麻烦

## 1.2 限制

- Pod的存储必须使用 PersistVolume
- 删除或者缩容时，不会删除关联的卷
- 使用 headless service 关联 Pod，需要手动创建
- Pod 的管理策略为 `OrderedReady` 时使用滚动更新能力，可能需要人工干预

## 1.3 基本功能
- 创建
- 删除
    - 级联删除
    - 非级连删除
- 扩容/缩容：扩容为顺序，缩容则为逆运算，即逆序
- 更新：与无状态应用不同的是，StatefulSet 是基于 `ControllerRevision` 保存更新记录
    - `RollingUpdate`
    - `OnDelete`
- Pod 管理策略：对于某些分布式系统来说，StatefulSet 的顺序性保证是不必要的，所以引入了 `statefulset.spec.podManagementPolicy` 字段
    - `OrderedReady`
    - `Parallel`：允许StatefulSet Controller 并行终止所有 Pod，不必按顺序启动或删除 Pod

## 1.4 示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # headless service
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  podManagementPolicy: "OrderedReady" # default is OrderedReady
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

# 2. 源码解析

> kubernetes version: v1.19

## 2.1 startStatefulSetController()

`startStatefulSetController()` 是 StatefulSet Controller 的启动方法，其中调用 `statefulset.NewStatefulSetController()` 方法进行初始化，然后调用对象的 `Run()` 方法启动 Controller。其中 `ConcurrentStatefulSetSyncs` 默认是5，即默认启动 5 个协程处理StatefulSet 相关业务。

可以看到 StatefulSetController 初始化时直接相关的对象类型分别是 Pod、StatefulSet、PVC 和 ControllerRevision。印证了之前提到的 StatefulSet 的特殊之处：使用 PV 作为存储；ControllerRevision 表示升级/回滚记录。

```go
func startStatefulSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "statefulsets"}] {
		return nil, false, nil
	}
	go statefulset.NewStatefulSetController(
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Apps().V1().StatefulSets(),
		ctx.InformerFactory.Core().V1().PersistentVolumeClaims(),
		ctx.InformerFactory.Apps().V1().ControllerRevisions(),
		ctx.ClientBuilder.ClientOrDie("statefulset-controller"),
	).Run(int(ctx.ComponentConfig.StatefulSetController.ConcurrentStatefulSetSyncs), ctx.Stop)
	return nil, true, nil
}
```

## 2.2 sync()

`run()` 方法会通过 informer 同步 cache 并监听 pod、statefulset、pvc 和 controllerrevision 对象的变更事件，然后启动 5 个 worker 协程，每个 worker 调用 `sync()` 方法，正式进入业务逻辑处理。

```go
func (ssc *StatefulSetController) sync(key string) error {
	...
    // 1. 解析 namespace 和 name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	...
    
    // 2. 根据 ns 和 name，获取 sts 对象
	set, err := ssc.setLister.StatefulSets(namespace).Get(name)
	...

    // 3. 获取 selector 对象，用于筛选 Pod
	selector, err := metav1.LabelSelectorAsSelector(set.Spec.Selector)
	...

    // ssc.adoptOrphanRevisions() 
    // -> ssc.control.AdoptOrphanRevisions() 
    // -> ssc.controllerHistory.AdoptControllerRevision() 
    // -> Patch()
    // 4、筛选 sts 的孤儿 controllerrevisions，并尝试与 sts 重新关联（添加 ControllerRef）
	if err := ssc.adoptOrphanRevisions(set); err != nil {
		return err
	}

    // 5. 获取 sts 所有关联的 pod
    // ssc.getPodsForStatefulSet() 
    // -> cm.ClaimPods() 
    // -> m.ClaimObject() 
    // -> release()/adopt()
	pods, err := ssc.getPodsForStatefulSet(set, selector)
	...

    // 6. 真正执行 sync 操作
	return ssc.syncStatefulSet(set, pods)
}
```

则，`sync()` 的主要逻辑为：
1. 根据 ns/name 获取 sts 对象；
2. 获取 sts 的 `selector`；
3. 调用 `ssc.adoptOrphanRevisions()` 检查是否有孤儿 controllerrevisions 对象，若有且能匹配 selector 的则添加 ownerReferences 进行关联；
4. 调用 `ssc.getPodsForStatefulSet` 通过 selector 获取 sts 关联的 pod，若有孤儿 pod 的 label 与 sts 的能匹配则进行关联，若已关联的 pod label 有变化则解除与 sts 的关联关系；
5. 最后调用 ssc.syncStatefulSet 执行真正的 sync 操作；

## 2.3 syncStatefulSet()

在 `syncStatefulSet()` 中仅仅是调用了 `ssc.control.UpdateStatefulSet()` 方法进行处理。`ssc.control.UpdateStatefulSet()` 会调用 `ssc.performUpdate()` 方法，最终走到更新逻辑 `ssc.updateStatefulSet()` 和 `ssc.updateStatefulSetStatus()`。

```go
func (ssc *StatefulSetController) syncStatefulSet(set *apps.StatefulSet, pods []*v1.Pod) error {
	...
    // ssc.control.UpdateStatefulSet() 
    // -> ssc.performUpdate()
    // -> ssc.updateStatefulSet() + ssc.updateStatefulSetStatus()
	if err := ssc.control.UpdateStatefulSet(set.DeepCopy(), pods); err != nil {
		return err
	}
	...
	return nil
}

func (ssc *defaultStatefulSetControl) UpdateStatefulSet(set *apps.StatefulSet, pods []*v1.Pod) error {

	// 1. 获取历史 revisions，并排序
	revisions, err := ssc.ListRevisions(set)
	...
	history.SortControllerRevisions(revisions)
    
    // 2. 计算 currentRevision 和 updateRevision
	currentRevision, updateRevision, err := ssc.performUpdate(set, pods, revisions)
	if err != nil {
		return utilerrors.NewAggregate([]error{err, ssc.truncateHistory(set, pods, revisions, currentRevision, updateRevision)})
	}

	// 3. 清理过期的历史版本
	return ssc.truncateHistory(set, pods, revisions, currentRevision, updateRevision)
}

func (ssc *defaultStatefulSetControl) performUpdate(
	set *apps.StatefulSet, pods []*v1.Pod, revisions []*apps.ControllerRevision) (*apps.ControllerRevision, *apps.ControllerRevision, error) {

	// 2.1. 计算 currentRevision 和 updateRevision
	currentRevision, updateRevision, collisionCount, err := ssc.getStatefulSetRevisions(set, revisions)
	if err != nil {
		return currentRevision, updateRevision, err
	}

	// 2.2. 执行实际的 sync 操作
	status, err := ssc.updateStatefulSet(set, currentRevision, updateRevision, collisionCount, pods)
	if err != nil {
		return currentRevision, updateRevision, err
	}

	// 2.3. 更新 sts 状态
	err = ssc.updateStatefulSetStatus(set, status)
	if err != nil {
		return currentRevision, updateRevision, err
	}

	...
	return currentRevision, updateRevision, nil
}
```

整体来看，ssc.control.UpdateStatefulSet() 方法的主要逻辑为：
1. 获取历史 revisions；
2. 计算 currentRevision 和 updateRevision，若 sts 处于更新过程中则 currentRevision 和 updateRevision 值不同；
3. 调用 `ssc.updateStatefulSet()` 执行实际的 sync 操作；
4. 调用 `ssc.updateStatefulSetStatus()` 更新 status 子资源；
5. 根据 sts 的 spec.revisionHistoryLimit 字段清理过期的 controllerrevision； 

## 2.4 ssc.updateStatefulSet()
sts 通过 controllerrevision 保存历史版本，类似于 deployment 的 replicaset，与 replicaset 不同的是 controllerrevision 仅用于回滚阶段，在 sts 的滚动升级过程中是通过 currentRevision 和 updateRevision 进行控制并不会用到 controllerrevision。

```go
func (ssc *defaultStatefulSetControl) updateStatefulSet(...) (*apps.StatefulSetStatus, error) {
	// 1. 分别获取 currentRevision 和 updateRevision 对应的的 statefulset object
	currentSet, err := ApplyRevision(set, currentRevision)
	...
	updateSet, err := ApplyRevision(set, updateRevision)
	...

	// 设置 status 的 generation 和 revisions
	status := apps.StatefulSetStatus{}
	status.ObservedGeneration = set.Generation
	status.CurrentRevision = currentRevision.Name
	status.UpdateRevision = updateRevision.Name
	status.CollisionCount = new(int32)
	*status.CollisionCount = collisionCount

    // 3. 将 statefulset 的 pods 按序分到 replicas 和 condemned 两个切片
	replicaCount := int(*set.Spec.Replicas)
    // 用于保存序号在 [0,replicas) 范围内的 pod
	replicas := make([]*v1.Pod, replicaCount)
	// 用于序号大于 replicas 的 Pod
	condemned := make([]*v1.Pod, 0, len(pods))
	unhealthy := 0
	firstUnhealthyOrdinal := math.MaxInt32
	var firstUnhealthyPod *v1.Pod

    // 4. 计算 status 字段中的值，将 pod 分配到 replicas 和 condemned两个数组中
	for i := range pods {
		status.Replicas++

		// 计算 Ready 的副本数
		if isRunningAndReady(pods[i]) {
			status.ReadyReplicas++
		}

		// 计算当前的副本数和已经更新的副本数
		if isCreated(pods[i]) && !isTerminating(pods[i]) {
			if getPodRevision(pods[i]) == currentRevision.Name {
				status.CurrentReplicas++
			}
			if getPodRevision(pods[i]) == updateRevision.Name {
				status.UpdatedReplicas++
			}
		}

		if ord := getOrdinal(pods[i]); 0 <= ord && ord < replicaCount {
			// 保存序号在 [0,replicas) 范围内的 Pod
			replicas[ord] = pods[i]

		} else if ord >= replicaCount {
			// 序号大于 replicas 的 Pod，则保存在 condemned
			condemned = append(condemned, pods[i])
		}
		// 其余的 Pod 忽略
	}

    // 5. 检查 replicas数组中 [0,set.Spec.Replicas) 下标是否有缺失的 pod，若有缺失的则创建对应的 Pod 
	// 在 newVersionedStatefulSetPod 中会判断是使用 currentSet 还是 updateSet 来创建
	for ord := 0; ord < replicaCount; ord++ {
		if replicas[ord] == nil {
			replicas[ord] = newVersionedStatefulSetPod(
				currentSet,
				updateSet,
				currentRevision.Name,
				updateRevision.Name, ord)
		}
	}

	// 6. 对 condemned 数组进行排序
	sort.Sort(ascendingOrdinal(condemned))

    // 7、根据 ordinal 在 replicas 和 condemned 数组中找出第一个处于 NotReady 或 Terminating 的 Pod
	for i := range replicas {
		if !isHealthy(replicas[i]) {
			unhealthy++
			if ord := getOrdinal(replicas[i]); ord < firstUnhealthyOrdinal {
				firstUnhealthyOrdinal = ord
				firstUnhealthyPod = replicas[i]
			}
		}
	}

	for i := range condemned {
		if !isHealthy(condemned[i]) {
			unhealthy++
			if ord := getOrdinal(condemned[i]); ord < firstUnhealthyOrdinal {
				firstUnhealthyOrdinal = ord
				firstUnhealthyPod = condemned[i]
			}
		}
	}

	if unhealthy > 0 {
		klog.V(4).Infof("StatefulSet %s/%s has %d unhealthy Pods starting with %s",
			set.Namespace,
			set.Name,
			unhealthy,
			firstUnhealthyPod.Name)
	}

	// 8. 如果 sts 正在删除中，只要更新 status 即可
	if set.DeletionTimestamp != nil {
		return &status, nil
	}

    // 9. 默认设置为非 Parallel
	monotonic := !allowsBurst(set)

    // 10. 确保 replicas 数组中所有的 pod 是 running 的
	for i := range replicas {
		// 11. 对于 failed 的 pod 删除并重建
		if isFailed(replicas[i]) {
			ssc.recorder.Eventf(set, v1.EventTypeWarning, "RecreatingFailedPod",
				"StatefulSet %s/%s is recreating failed Pod %s",
				set.Namespace,
				set.Name,
				replicas[i].Name)
           // 删除
			if err := ssc.podControl.DeleteStatefulPod(set, replicas[i]); err != nil {
				return &status, err
			}
			if getPodRevision(replicas[i]) == currentRevision.Name {
				status.CurrentReplicas--
			}
			if getPodRevision(replicas[i]) == updateRevision.Name {
				status.UpdatedReplicas--
			}
			status.Replicas--
           // 新建
			replicas[i] = newVersionedStatefulSetPod(
				currentSet,
				updateSet,
				currentRevision.Name,
				updateRevision.Name,
				i)
		}
		// 12、如果 pod.Status.Phase 不为空，说明该 pod 未创建，则直接重新创建该 pod
		if !isCreated(replicas[i]) {
			if err := ssc.podControl.CreateStatefulPod(set, replicas[i]); err != nil {
				return &status, err
			}
			status.Replicas++
			if getPodRevision(replicas[i]) == currentRevision.Name {
				status.CurrentReplicas++
			}
			if getPodRevision(replicas[i]) == updateRevision.Name {
				status.UpdatedReplicas++
			}

			// 13. 如果为 Parallel，直接 return status 结束；如果为 OrderedReady，循环处理下一个pod。
			if monotonic {
				return &status, nil
			}
			// pod 创建完成，本轮循环结束
			continue
		}
        // 14、如果 pod 正在删除中，且 Spec.PodManagementPolicy 不为 Parallel，
        // 直接return status结束，结束后会在下一个 syncLoop 继续进行处理，pod 状态的改变会触发下一次 syncLoop
		if isTerminating(replicas[i]) && monotonic {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to Terminate",
				set.Namespace,
				set.Name,
				replicas[i].Name)
			return &status, nil
		}
        // 15. 如果 pod 状态不是 Running & Ready，且 Spec.PodManagementPolicy 不为 Parallel
        // 则直接return status结束
		if !isRunningAndReady(replicas[i]) && monotonic {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to be Running and Ready",
				set.Namespace,
				set.Name,
				replicas[i].Name)
			return &status, nil
		}
		// 16. 检查 pod 的信息是否与 statefulset 的匹配，若不匹配则更新 pod 的状态
		if identityMatches(set, replicas[i]) && storageMatches(set, replicas[i]) {
			continue
		}
		// 深拷贝对象，避免修改缓存
		replica := replicas[i].DeepCopy()
		if err := ssc.podControl.UpdateStatefulPod(updateSet, replica); err != nil {
			return &status, err
		}
	}

    // 17. 逆序处理 condemned 中的 pod
	for target := len(condemned) - 1; target >= 0; target-- {
        // 18. 如果 pod 正在删除，检查 Spec.PodManagementPolicy 的值
        // 如果为 Parallel，循环处理下一个pod 否则直接退出
		if isTerminating(condemned[target]) {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to Terminate prior to scale down",
				set.Namespace,
				set.Name,
				condemned[target].Name)
			// 如果不为 Parallel，直接 return
			if monotonic {
				return &status, nil
			}
			continue
		}
		// 19. 不满足以下条件说明该 pod 是更新前创建的，正处于创建中
		if !isRunningAndReady(condemned[target]) && monotonic && condemned[target] != firstUnhealthyPod {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to be Running and Ready prior to scale down",
				set.Namespace,
				set.Name,
				firstUnhealthyPod.Name)
			return &status, nil
		}
		klog.V(2).Infof("StatefulSet %s/%s terminating Pod %s for scale down",
			set.Namespace,
			set.Name,
			condemned[target].Name)
        // 20、否则直接删除该 pod
		if err := ssc.podControl.DeleteStatefulPod(set, condemned[target]); err != nil {
			return &status, err
		}
		if getPodRevision(condemned[target]) == currentRevision.Name {
			status.CurrentReplicas--
		}
		if getPodRevision(condemned[target]) == updateRevision.Name {
			status.UpdatedReplicas--
		}
        // 21. 如果为 OrderedReady 方式则返回否则继续处理下一个 pod
		if monotonic {
			return &status, nil
		}
	}

	// 22. 对于 OnDelete 策略直接返回
	if set.Spec.UpdateStrategy.Type == apps.OnDeleteStatefulSetStrategyType {
		return &status, nil
	}

	// 23. 若为 RollingUpdate 策略，则倒序处理 replicas 数组中下标大于等于
    // 	   Spec.UpdateStrategy.RollingUpdate.Partition 的 pod
	updateMin := 0
	if set.Spec.UpdateStrategy.RollingUpdate != nil {
		updateMin = int(*set.Spec.UpdateStrategy.RollingUpdate.Partition)
	}
	// 用与更新版本不匹配的最大序号终止 Pod
	for target := len(replicas) - 1; target >= updateMin; target-- {

		// 24. 如果 Pod 的 Revision 不等于 updateRevision，且 pod 没有处于删除状态，则直接删除 pod
		if getPodRevision(replicas[target]) != updateRevision.Name && !isTerminating(replicas[target]) {
			klog.V(2).Infof("StatefulSet %s/%s terminating Pod %s for update",
				set.Namespace,
				set.Name,
				replicas[target].Name)
			err := ssc.podControl.DeleteStatefulPod(set, replicas[target])
			status.CurrentReplicas--
			return &status, err
		}

		// 25. 如果 pod 非 healthy 状态直接返回
		if !isHealthy(replicas[target]) {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to update",
				set.Namespace,
				set.Name,
				replicas[target].Name)
			return &status, nil
		}

	}
	return &status, nil
}
```

综上，`updateStatefulSet()` 把 statefulset 的创建、删除、更新、扩缩容的操作都包含在内。主要逻辑为：
1. 分别获取 `currentRevision` 和 `updateRevision` 所对应的 sts 对象
2. 取出 `sts.status` 并设置相关新值，用于更新
3. 将 sts 关联的 Pod，按照序号分到 `replicas` 和 `condemned` 两个切片中，replicas 保存的是序号在 [0, spec.replicas) 之间的 Pod，表示可用，condemned 保存序号大于 `spec.replicas` 的 Pod，表示待删除；
4. 找出 `replicas` 和 `condemned` 组中的 unhealthy pod，healthy pod 指 running & ready 并且不处于删除状态；
5. 判断 sts 是否处于删除状态；
6. 遍历 `replicas`，确保其中的 Pod 处于 running & ready 状态，其中处于 Failed 状态的 Pod 删除重建；未创建的容器则直接创建；处于删除中的，等待优雅删除结束，即下一轮循环再处理；最后检查 pod 的信息是否与 statefulset 的匹配，若不匹配则更新 pod。在此过程中每一步操作都会检查 `.Spec.podManagementPolicy` 是否为 `Parallel`，若设置了则循环处理 replicas 中的所有 pod，否则每次处理一个 pod，剩余 pod 则在下一个 syncLoop 继续进行处理；
7. 按 pod 名称逆序删除 condemned 数组中的 pod，删除前也要确保 pod 处于 running & ready 状态，在此过程中也会检查 `.Spec.podManagementPolicy` 是否为 `Parallel`，以此来判断是顺序删除还是在下一个 syncLoop 中继续进行处理；
8. 判断 sts 的更新策略 `.Spec.UpdateStrategy.Type`，若为 OnDelete 则直接返回；
9. 此时更新策略为 `RollingUpdate`，更新序号大于等于 `.Spec.UpdateStrategy.RollingUpdate.Partition` 的 pod；更新策略为 `RollingUpdate`，并不会关注 `.Spec.podManagementPolicy`，都是顺序进行处理，且等待当前 pod 删除成功后才继续逆序删除一下 pod，所以 Parallel 的策略在滚动更新时无法使用。

`updateStatefulSet()` 这个方法中包含了 statefulset 的创建、删除、扩缩容、更新等操作，在源码层面对于各个功能无法看出明显的界定，没有 deployment sync 方法中写的那么清晰，下面按 statefulset 的功能再分析一下具体的操作：
- 创建：在创建 sts 后，sts 对象已被保存至 etcd 中，此时 sync 操作仅仅是创建出需要的 pod，即执行到第 6 步就会结束；
- 扩缩容：对于扩若容操作仅仅是创建或者删除对应的 pod，在操作前也会判断所有 pod 是否处于 running & ready 状态，然后进行对应的创建/删除操作，在上面的步骤中也会执行到第 6 步就结束；
- 更新：可以看出在第 6 步之后的所有操作就是与更新相关，所以更新操作会执行完整个方法，在更新过程中通过 pod 的 currentRevision 和 updateRevision 来计算 currentReplicas、updatedReplicas 的值，最终完成所有 pod 的更新；
- 删除：删除操作就比较明显，会止于第 5 步，但是在此之前检查 pod 状态以及分组的操作确实是多余的；


# 3. 参考链接

- [StatefulSet 概念](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/)
- [StatefulSet 基础](https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/)
- [StatefulSet 源码解析](https://cloud.tencent.com/developer/article/1554676)