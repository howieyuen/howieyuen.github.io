---
author: Yuan Hao
date: 2020-10-21
title: DaemonSet Controller 源码分析
tag: [daemonset, controller]
weight: 1
---

# 1. DaemonSet 简介

我们知道，Deployment 是用来部署一定数量的 Pod。但是，当你希望 Pod 在集群中的每个节点上运行，并且每个节点上都需要一个 Pod 实例时，Deployment 就无法满足需求。

这类需求包括 Pod 执行系统级别与基础结构相关的操作，比如：希望在每个节点上运行日志收集器和资源监控组件。另一个典型的例子，就是 Kubernetes 自己的 kube-proxy 进程，它需要在所有节点上都运行，才能使得 Service 正常工作。

如此，DaemonSet 应运而生。它能确保集群中每个节点或者是满足某些特性的一组节点都运行一个 Pod 副本。当有新节点加入时，也会立即为它部署一个 Pod；当有节点从集群中删除时，Pod 也会被回收。删除 DaemonSet，也会删除所有关联的 Pod。

## 1.1 应用场景

- 在每个节点上运行集群存守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

一种简单的用法是为每种类型的守护进程在所有的节点上都启动一个 DaemonSet。 一个稍微复杂的用法是为同一种守护进程部署多个 DaemonSet；每个具有不同的标志，并且对不同硬件类型具有不同的内存、CPU 等要求。

## 1.2 基本功能

- 创建
- 删除
    - 级联删除：`kubectl delete ds/nginx-ds`
    - 非级联删除：`kubectl delete ds/nginx-ds --cascade=false`
- 更新
    - RollingUpdate
    - OnDelete
- 回滚

## 1.3 示例

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

# 2. 源码分析

> kubernetes version: v1.19

## 2.1 startDaemonSetController()

与其他资源的 Controller 启动方式一致，在 `startDaemonSetController()` 中初始化 DaemonSetController 对象，并调用 `Run()` 方法启动。从该方法可以看出，DaemonSet Controller 关心的是 `daemonset`、`controllerrevision`、`pod`、`node` 四种资源的变动，其中 `ConcurrentDaemonSetSyncs`  默认是 2。

`cmd/kube-controller-manager/app/apps.go:36`

```go
func startDaemonSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "daemonsets"}] {
		return nil, false, nil
	}
	dsc, err := daemon.NewDaemonSetsController(
		ctx.InformerFactory.Apps().V1().DaemonSets(),
		ctx.InformerFactory.Apps().V1().ControllerRevisions(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Core().V1().Nodes(),
		ctx.ClientBuilder.ClientOrDie("daemon-set-controller"),
		flowcontrol.NewBackOff(1*time.Second, 15*time.Minute),
	)
	if err != nil {
		return nil, true, fmt.Errorf("error creating DaemonSets controller: %v", err)
	}
	go dsc.Run(int(ctx.ComponentConfig.DaemonSetController.ConcurrentDaemonSetSyncs), ctx.Stop)
	return nil, true, nil
}
```

## 2.2 Run()

Run() 方法中执行 2 个核心操作：`sync` 和 `gc`。其中 sync 操作是 controller 的核心代码，响应上述所有操作。在初始化 controller 对象时，指定了 `failedPodsBackoff` 的参数，`defaultDuration = 1s`，`maxDuration = 15min`；`gc` 的主要作用是发现 daemonset 的 pod 的 phase 为 failed，就会重启该 Pod，如果已经超时（2*15min）会删除该条记录。

`pkg/controller/daemon/daemon_controller.go:281`

```go
func (dsc *DaemonSetsController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer dsc.queue.ShutDown()

	klog.Infof("Starting daemon sets controller")
	defer klog.Infof("Shutting down daemon sets controller")

	if !cache.WaitForNamedCacheSync("daemon sets", stopCh, dsc.podStoreSynced, dsc.nodeStoreSynced, dsc.historyStoreSynced, dsc.dsStoreSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		// sync
		go wait.Until(dsc.runWorker, time.Second, stopCh)
	}
	// gc
	go wait.Until(dsc.failedPodsBackoff.GC, BackoffGCInterval, stopCh)

	<-stopCh
}
```

## 2.3 syncDaemonSet()

DaemonSet 的 Pod 的创建/删除都是和 Node 相关，所以每次 `sync` 操作，需要遍历所有的 Node 进行判断。`syncDaemonSet()` 的主要逻辑为：

1. 通过 key 获取 ns 和 name 
2. 从 dsLister 获取 ds 对象
3. 从 nodeLister 获取全部 node 对象
4. 获取 dsKey， 即：`<namespace>/<name>`
5. 判断 ds 是否处于删除中，如果正在删除，则等待删除完毕后再次进入 sync
6. 获取 cur 和 old `controllerrevision`
7. 判断是否满足 `expectation` 机制，`expectation` 机制就是为了减少不必要的 `sync` 操作
8. 调用 `dsc.manage()`，执行实际的 sync  操作
9. 判断是否为更新操作，并执行
10. 调用 `dsc.cleanupHistory()` 根据 `spec.revisionHistroyLimit` 清理过期的 `controllerrevision`
11. 调用 `dsc.updateDaemonSetStatus()`，更新 status 子资源

`pkg/controller/daemon/daemon_controller.go:1129`

```go
func (dsc *DaemonSetsController) syncDaemonSet(key string) error {
	...
    
	// 1. 通过 key 获取 ns 和 name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	...
    
	// 2. 从 dsLister 获取 ds 对象
	ds, err := dsc.dsLister.DaemonSets(namespace).Get(name)
	...

	// 3. 从 nodeLister 获取全部 node 对象
	nodeList, err := dsc.nodeLister.List(labels.Everything())
	...
    
	// 4. 获取 dsKey， 即：<namespace>/<name>
	dsKey, err := controller.KeyFunc(ds)
	...
	
	// 5. 判断 ds 是否处于删除中，如果正在删除，则等待删除完毕后再次进入 sync
	if ds.DeletionTimestamp != nil {
		return nil
	}

	// 6. 获取 cur 和 old controllerrevision
	cur, old, err := dsc.constructHistory(ds)
	// hash 就是当前 controllerrevision 的 controller-revision-hash 值
	hash := cur.Labels[apps.DefaultDaemonSetUniqueLabelKey]

	// 7. 判断是否满足 expectation 机制
	if !dsc.expectations.SatisfiedExpectations(dsKey) {
		// 不满足，只更新 status 子资源
		return dsc.updateDaemonSetStatus(ds, nodeList, hash, false)
	}

	// 8. 实际执行的 sync  操作
	err = dsc.manage(ds, nodeList, hash)
	...

	// 9. 判断是否为更新操作，并执行
	if dsc.expectations.SatisfiedExpectations(dsKey) {
		switch ds.Spec.UpdateStrategy.Type {
		case apps.OnDeleteDaemonSetStrategyType:
		case apps.RollingUpdateDaemonSetStrategyType:
			err = dsc.rollingUpdate(ds, nodeList, hash)
		}
		if err != nil {
			return err
		}
	}

	// 10. 清理过期的 controllerrevision
	err = dsc.cleanupHistory(ds, old)
	if err != nil {
		return fmt.Errorf("failed to clean up revisions of DaemonSet: %v", err)
	}

	// 11. 最后更新 status 子资源
	return dsc.updateDaemonSetStatus(ds, nodeList, hash, true)
}
```

## 2.4 dsc.manage()

`dsc.manage()` 是为了保证ds 的 Pod 正常运行在应该存在的节点，该方法做了这几件事：
1. 调用 `dsc.getNodesToDaemonPods()`，获取当前的节点和 daemon pod 的映射关系（`map[nodeName][]*v1.Pod`）
2. 遍历所有节点，判断每个节点是需要创建还是删除 daemonset pod
3. 从 1.12 开始，daemon pod 已经由 `kube-scheduler` 负责调度，可能会出现把 daemon pod 调度到不存在的节点上，如果存在这种情况，就要删除该 pod
4. 调用 `dsc.syncNodes()` 为对应的 node 创建 daemon pod 以及删除多余的 pods；

`pkg/controller/daemon/daemon_controller.go:881`
```go
func (dsc *DaemonSetsController) manage(ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
    // 1. 找到节点和 ds 创建的 Pod 的映射关系（nodeName:[]Pod{}）
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ds)
	...

	// 2. 检查每个节点是否应当运行该 daemonset 的 Pod，如果不该运行就要删掉，反之就要创建
	var nodesNeedingDaemonPods, podsToDelete []string
	for _, node := range nodeList {
		nodesNeedingDaemonPodsOnNode, podsToDeleteOnNode, err := dsc.podsShouldBeOnNode(
			node, nodeToDaemonPods, ds)

		if err != nil {
			continue
		}

		nodesNeedingDaemonPods = append(nodesNeedingDaemonPods, nodesNeedingDaemonPodsOnNode...)
		podsToDelete = append(podsToDelete, podsToDeleteOnNode...)
	}

    // 3. 调用 getUnscheduledPodsWithoutNode() 方法找到把调度到不存在的节点的 Pod
	podsToDelete = append(podsToDelete, getUnscheduledPodsWithoutNode(nodeList, nodeToDaemonPods)...)

    // 4. 调用 dsc.syncNodes()，删除多余的Pod，为应该运行 daemonset Pod 的 node 创建Pod
	if err = dsc.syncNodes(ds, podsToDelete, nodesNeedingDaemonPods, hash); err != nil {
		return err
	}

	return nil
}
```

下面继续看下 `dsc.podsShouldBeOnNode()` 和 `dsc.syncNodes()` 两个方法的具体逻辑：

其中，`podsShouldBeOnNode()` 主要是为了确定在给定节点上的是需要创建还是删除 daemon pod。主要逻辑为：
1. 调用 `dsc.nodeShouldRunDaemonPod` 判断该 node 是否要运行以及是否能继续运行 ds pod
2. 获取该节点上的 daemon pod 列表
3. 根据 `shouldRun`、`shouldContinueRunning` 和 `exists`（daemon pod 的存在状态），进行下一步
    1. 如果节点应该运行而没有运行，则创建该 Pod
    2. 如果 daemon pods 应该继续在此节点上运行，遍历每个 daemon pod，且
        1. pod 在删除中，暂且不管
        2. pod 处于 failed 状态，则删除 pod
        3. daemon pods 数量 > 1，则保留最早创建的 pod，其余都删除
    3. 如果 pod 不需要继续运行但 pod 已存在，则需要删除
4. 最终返回需要运行 daemon pod 的节点集合和待删除 pod 的集合

```go
func (dsc *DaemonSetsController) podsShouldBeOnNode(
	node *v1.Node,
	nodeToDaemonPods map[string][]*v1.Pod,
	ds *apps.DaemonSet,
) (nodesNeedingDaemonPods, podsToDelete []string, err error) {
	// 1. 判断该 node 是否要运行以及是否能继续运行 ds pod
	shouldRun, shouldContinueRunning, err := dsc.nodeShouldRunDaemonPod(node, ds)
	if err != nil {
		return
	}
	// 2. 获取该节点上的该 daemon pod 列表
	daemonPods, exists := nodeToDaemonPods[node.Name]

	switch {
	case shouldRun && !exists:
		// 3.1 如果节点应该运行而没有运行，则创建该 Pod
		nodesNeedingDaemonPods = append(nodesNeedingDaemonPods, node.Name)
	case shouldContinueRunning:
		// 3.2. 如果 daemon pod应该继续在此节点上运行
		var daemonPodsRunning []*v1.Pod
		for _, pod := range daemonPods {
			// 3.2.1 pod 在删除中，暂且不管
			if pod.DeletionTimestamp != nil {
				continue
			}
			// 3.2.2 pod 处于 failed 状态，则删除 pod
			if pod.Status.Phase == v1.PodFailed {
				// This is a critical place where DS is often fighting with kubelet that rejects pods.
				// We need to avoid hot looping and backoff.
				backoffKey := failedPodsBackoffKey(ds, node.Name)

				now := dsc.failedPodsBackoff.Clock.Now()
				inBackoff := dsc.failedPodsBackoff.IsInBackOffSinceUpdate(backoffKey, now)
				if inBackoff {
					delay := dsc.failedPodsBackoff.Get(backoffKey)
					klog.V(4).Infof("Deleting failed pod %s/%s on node %s has been limited by backoff - %v remaining",
						pod.Namespace, pod.Name, node.Name, delay)
					dsc.enqueueDaemonSetAfter(ds, delay)
					continue
				}

				dsc.failedPodsBackoff.Next(backoffKey, now)

				msg := fmt.Sprintf("Found failed daemon pod %s/%s on node %s, will try to kill it", pod.Namespace, pod.Name, node.Name)
				klog.V(2).Infof(msg)
				// Emit an event so that it's discoverable to users.
				dsc.eventRecorder.Eventf(ds, v1.EventTypeWarning, FailedDaemonPodReason, msg)
				podsToDelete = append(podsToDelete, pod.Name)
			} else {
				daemonPodsRunning = append(daemonPodsRunning, pod)
			}
		}
		// 3.2.3 如果 daemon pod 应该在此节点上运行，但存在的pod数 > 1，则保留最早创建的pod，其余删除
		if len(daemonPodsRunning) > 1 {
			sort.Sort(podByCreationTimestampAndPhase(daemonPodsRunning))
			for i := 1; i < len(daemonPodsRunning); i++ {
				podsToDelete = append(podsToDelete, daemonPodsRunning[i].Name)
			}
		}
	case !shouldContinueRunning && exists:
		// 3.3 如果 pod 不需要继续运行但 pod 已存在则需要删除 pod
		for _, pod := range daemonPods {
			if pod.DeletionTimestamp != nil {
				continue
			}
			podsToDelete = append(podsToDelete, pod.Name)
		}
	}
    // 4. 最终返回需要运行 daemon pod 的节点集合和待删除 pod 集合
	return nodesNeedingDaemonPods, podsToDelete, nil
}
```

继续看 `dsc.syncNode()`，该方法主要是为需要 daemon pod 的 node 创建 pod 以及删除多余的 pod，其主要逻辑为：
1. 将 `createDiff` 和 `deleteDiff` 与 `burstReplicas` 进行比较，`burstReplicas` 默认值为 250，即每个 syncLoop 中创建或者删除的 pod 数最多为 250 个，若超过其值则剩余需要创建或者删除的 pod 在下一个 syncLoop 继续操作；
2. 将 `createDiff` 和 `deleteDiff` 写入到 `expectations` 中；
3. 并发创建 pod，通过 `nodeAffinity` 来保证每个节点都运行一个 pod；
4. 并发删除 `deleteDiff` 中的所有 pod；

```go
func (dsc *DaemonSetsController) syncNodes(ds *apps.DaemonSet, podsToDelete, nodesNeedingDaemonPods []string, hash string) error {
	// 1. 取 ds 的 key (namespace/name)
	dsKey, err := controller.KeyFunc(ds)
	...
	// 2. 设置创删过程中可超过的副本数
	createDiff := len(nodesNeedingDaemonPods)
	deleteDiff := len(podsToDelete)

	if createDiff > dsc.burstReplicas {
		createDiff = dsc.burstReplicas
	}
	if deleteDiff > dsc.burstReplicas {
		deleteDiff = dsc.burstReplicas
	}

	// 3. 设置 expectation
	dsc.expectations.SetExpectations(dsKey, createDiff, deleteDiff)

	// error channel to communicate back failures.  make the buffer big enough to avoid any blocking
	errCh := make(chan error, createDiff+deleteDiff)

	createWait := sync.WaitGroup{}
	
	generation, err := util.GetTemplateGeneration(ds)
	...
	template := util.CreatePodTemplate(ds.Spec.Template, generation, hash)
	
	// 4. 并发创建 pod，创建的 pod 数依次为 1, 2, 4, 8, ...
	batchSize := integer.IntMin(createDiff, controller.SlowStartInitialBatchSize)
	for pos := 0; createDiff > pos; batchSize, pos = integer.IntMin(2*batchSize, createDiff-(pos+batchSize)), pos+batchSize {
		errorCount := len(errCh)
		createWait.Add(batchSize)
		for i := pos; i < pos+batchSize; i++ {
			go func(ix int) {
				defer createWait.Done()

				podTemplate := template.DeepCopy()
				
				// 5. 使用节点亲和性完成 daemon pod 调度
				podTemplate.Spec.Affinity = util.ReplaceDaemonSetPodNodeNameNodeAffinity(
					podTemplate.Spec.Affinity, nodesNeedingDaemonPods[ix])

				// 6. 创建Pod 带上 ControllerRef
				err := dsc.podControl.CreatePodsWithControllerRef(ds.Namespace, podTemplate,
					ds, metav1.NewControllerRef(ds, controllerKind))

				if err != nil {
					if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
						// 如果此时namespace被删除，这里错误可以忽略
						return
					}
				}
				if err != nil {
					klog.V(2).Infof("Failed creation, decrementing expectations for set %q/%q", ds.Namespace, ds.Name)
					dsc.expectations.CreationObserved(dsKey)
					errCh <- err
					utilruntime.HandleError(err)
				}
			}(i)
		}
		createWait.Wait()
	
		// 6. 将创建失败的 Pod 记录到 expectation 中
		skippedPods := createDiff - (batchSize + pos)
		if errorCount < len(errCh) && skippedPods > 0 {
			klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for set %q/%q", skippedPods, ds.Namespace, ds.Name)
			dsc.expectations.LowerExpectations(dsKey, skippedPods, 0)
			// skippedPod 会在下轮 sync 时重试
			break
		}
	}

	klog.V(4).Infof("Pods to delete for daemon set %s: %+v, deleting %d", ds.Name, podsToDelete, deleteDiff)
	deleteWait := sync.WaitGroup{}
	deleteWait.Add(deleteDiff)
	for i := 0; i < deleteDiff; i++ {
		// 7. 并发删除 deleteDiff 中的 pod
		go func(ix int) {
			defer deleteWait.Done()
			if err := dsc.podControl.DeletePod(ds.Namespace, podsToDelete[ix], ds); err != nil {
				dsc.expectations.DeletionObserved(dsKey)
				if !apierrors.IsNotFound(err) {
					klog.V(2).Infof("Failed deletion, decremented expectations for set %q/%q", ds.Namespace, ds.Name)
					errCh <- err
					utilruntime.HandleError(err)
				}
			}
		}(i)
	}
	deleteWait.Wait()

	// 收集错误返回
	errors := []error{}
	close(errCh)
	for err := range errCh {
		errors = append(errors, err)
	}
	return utilerrors.NewAggregate(errors)
}
```
到此，总结一下，`dsc.manage()` 方法的主要流程：

{{< mermaid >}}
graph LR
    op1(dsc.manage) -->  op2(dsc.getNodesToDaemonPods)
    op1 --> op3(dsc.podsShouldBeOnNode)
    op1 --> op4(dsc.syncNodes)
    op3 --> op5(dsc.nodeShouldRunDaemonPod)
{{< /mermaid >}}

## 2.5 dsc.rollingUpdate()

daemonset update 的方式有两种 `OnDelete` 和 `RollingUpdate`，当为 `OnDelete` 时需要用户手动删除每一个 pod 后完成更新操作，当为 `RollingUpdate` 时，daemonset controller 会自动控制升级进度。

当为 `RollingUpdate` 时，主要逻辑为：

1. 获取 node 与 daemon pods 的映射关系
2. 根据 `controllerrevision` 的 hash 值获取所有未更新的 pods；
3. 获取 `maxUnavailable`, `numUnavailable` 的 pod 数值，`maxUnavailable` 是从 ds 的 `rollingUpdate` 字段中获取的默认值为 1，`numUnavailable` 的值是通过 daemonset pod 与 node 的映射关系计算每个 node 下是否有 available pod 得到的；
4. 通过 `oldPods` 获取 `oldAvailablePods`, `oldUnavailablePods` 的 pod 列表；
5. 遍历 `oldUnavailablePods` 列表将需要删除的 pod 追加到 `oldPodsToDelete` 数组中。`oldUnavailablePods` 列表中的 pod 分为两种，一种处于更新中，即删除状态，一种处于未更新且异常状态，处于异常状态的都需要被删除；
6. 遍历 `oldAvailablePods` 列表，此列表中的 pod 都处于正常运行状态，根据 `maxUnavailable` 值确定是否需要删除该 pod 并将需要删除的 pod 追加到 `oldPodsToDelete` 数组中；
7. 调用 `dsc.syncNodes()` 删除 `oldPodsToDelete` 数组中的 pods，syncNodes 方法在 manage 阶段已经分析过，此处不再赘述；

`pkg/controller/daemon/update.go:44`
```go
func (dsc *DaemonSetsController) rollingUpdate(ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
	// 1. 获取 node 与 daemon pods 的映射关系
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	// 2. 获取所有未更新的 pods
	_, oldPods := dsc.getAllDaemonSetPods(ds, nodeToDaemonPods, hash)
	// 3. 计算 maxUnavailable, numUnavailable 的 pod 数值
	maxUnavailable, numUnavailable, err := dsc.getUnavailableNumbers(ds, nodeList, nodeToDaemonPods)
	if err != nil {
		return fmt.Errorf("couldn't get unavailable numbers: %v", err)
	}
	// 4. 把未更新的 pods 分成 available 和 unavailable
	oldAvailablePods, oldUnavailablePods := util.SplitByAvailablePods(ds.Spec.MinReadySeconds, oldPods)

	// 5. 将 unavailable 状态且没有删除标记的 pods 加入到 oldPodsToDelete 中
	var oldPodsToDelete []string
	klog.V(4).Infof("Marking all unavailable old pods for deletion")
	for _, pod := range oldUnavailablePods {
		// Skip terminating pods. We won't delete them again
		if pod.DeletionTimestamp != nil {
			continue
		}
		klog.V(4).Infof("Marking pod %s/%s for deletion", ds.Name, pod.Name)
		oldPodsToDelete = append(oldPodsToDelete, pod.Name)
	}

	// 6. 根据 maxUnavailable 值确定是否需要删除 pod
	klog.V(4).Infof("Marking old pods for deletion")
	for _, pod := range oldAvailablePods {
		if numUnavailable >= maxUnavailable {
			klog.V(4).Infof("Number of unavailable DaemonSet pods: %d, is equal to or exceeds allowed maximum: %d", numUnavailable, maxUnavailable)
			break
		}
		klog.V(4).Infof("Marking pod %s/%s for deletion", ds.Name, pod.Name)
		oldPodsToDelete = append(oldPodsToDelete, pod.Name)
		numUnavailable++
	}
	// 7. 调用 syncNodes 方法删除 oldPodsToDelete 数组中的 pods
	return dsc.syncNodes(ds, oldPodsToDelete, []string{}, hash)
}
```

## 2.6 dsc.updateDaemonSetStatus()

`dsc.updateDaemonSetStatus()` 是 sync 动作的最后一步，主要是用来更新 DaemonSet 的 status 子资源。ds.Status的各个字段如下：
```yaml
status:
  collisionCount: 0         # hash 冲突数
  currentNumberScheduled: 1  # 已经运行了 DaemonSet Pod 的节点数量
  desiredNumberScheduled: 1  # 需要运行该DaemonSet Pod的节点数量
  numberAvailable: 1        # DaemonSet Pod 状态为 Ready 且运行时间超过 Spec.MinReadySeconds 的节点数量
  numberUnavailable: 0      # desiredNumberScheduled - numberAvailable 的节点数量
  numberMisscheduled: 0     # 不需要运行 DeamonSet Pod 但是已经运行了的节点数量
  numberReady: 1           # DaemonSet Pod状态为Ready的节点数量
  observedGeneration: 1
  updatedNumberScheduled: 1 # 已经完成DaemonSet Pod更新的节点数量
```

主要逻辑为：

1. 调用 `dsc.getNodesToDaemonPods()` 获取node 与已存在的 daemon pods 的映射关系；
2. 遍历所有 node，调用 `dsc.nodeShouldRunDaemonPod()` 判断该 node 是否需要运行 daemon pod，然后计算 status 中的部分字段值；
3. 调用 `storeDaemonSetStatus()` 更新 ds.status；
4. 判断 ds 是否需要 resync；

`pkg/controller/daemon/daemon_controller.go:1075`
```go
func (dsc *DaemonSetsController) updateDaemonSetStatus(ds *apps.DaemonSet, nodeList []*v1.Node, hash string, updateObservedGen bool) error {
	klog.V(4).Infof("Updating daemon set status")
	// 1. 获取 node 与 daemon pods 的映射关系
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	var desiredNumberScheduled, currentNumberScheduled, numberMisscheduled, numberReady, updatedNumberScheduled, numberAvailable int
	for _, node := range nodeList {
		// 2. 判断该 node 是否需要运行 daemon pod
		shouldRun, _, err := dsc.nodeShouldRunDaemonPod(node, ds)
		if err != nil {
			return err
		}

		scheduled := len(nodeToDaemonPods[node.Name]) > 0
		
		// 3. 计算 status 中的字段值
		if shouldRun {
			desiredNumberScheduled++
			if scheduled {
				currentNumberScheduled++
				// 按照创建时间排序，最早创建在最前
				daemonPods, _ := nodeToDaemonPods[node.Name]
				sort.Sort(podByCreationTimestampAndPhase(daemonPods))
				pod := daemonPods[0]
				if podutil.IsPodReady(pod) {
					numberReady++
					if podutil.IsPodAvailable(pod, ds.Spec.MinReadySeconds, metav1.Now()) {
						numberAvailable++
					}
				}
				// If the returned error is not nil we have a parse error.
				// The controller handles this via the hash.
				generation, err := util.GetTemplateGeneration(ds)
				if err != nil {
					generation = nil
				}
				if util.IsPodUpdated(pod, hash, generation) {
					updatedNumberScheduled++
				}
			}
		} else {
			if scheduled {
				numberMisscheduled++
			}
		}
	}
	numberUnavailable := desiredNumberScheduled - numberAvailable

	// 4. 更新 ds.status
	err = storeDaemonSetStatus(dsc.kubeClient.AppsV1().DaemonSets(ds.Namespace), ds, desiredNumberScheduled, currentNumberScheduled, numberMisscheduled, numberReady, updatedNumberScheduled, numberAvailable, numberUnavailable, updateObservedGen)
	if err != nil {
		return fmt.Errorf("error storing status for daemon set %#v: %v", ds, err)
	}

	// Resync the DaemonSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	// 5. 判断 ds 是否需要 resync
	if ds.Spec.MinReadySeconds > 0 && numberReady != numberAvailable {
		dsc.enqueueDaemonSetAfter(ds, time.Duration(ds.Spec.MinReadySeconds)*time.Second)
	}
	return nil
}
```

## 2.7 源码分析小结

{{< mermaid >}}
graph LR
op(startDaeonSetController) --> op0(dsc.Run)
op0 --> op1(dsc.syncDaemonSet)
op1 --> op1.1(dsc.manage)
op1 --> op1.2(dsc.rollingUpdate)
op1 --> op1.3(dsc.updateDaemonSetStatus)
op1.1 --> op1.1.1(dsc.getNodesToDaemonPods)
op1.1 --> op1.1.2(dsc.podsShouldBeOnNode)
op1.1 --> op1.1.3(dsc.syncNodes)
op1.1.2 --> op1.1.2.1(dsc.nodeShouldRunDaemonPod)
{{< /mermaid >}}

# 3. 总结

在 daemonset controller 中可以看到许多功能都是 deployment 和 statefulset 已有的。在创建 pod 的流程与 replicaset controller 创建 pod 的流程是相似的，都使用了 `expectations` 机制并且限制了在一个 syncLoop 中最多创建或删除的 pod 数。更新方式与 statefulset 一样都有 `OnDelete` 和 `RollingUpdate` 两种， `OnDelete` 方式与 statefulset 相似，都需要手动删除对应的 pod，而  `RollingUpdate`  方式与 statefulset 和 deployment 都有点区别，`RollingUpdate` 方式更新时不支持暂停操作并且 pod 是先删除再创建的顺序进行。版本控制方式与 statefulset 的一样都是使用 `controllerRevision`。最后要说的一点是在 v1.12 及以后的版本中，使用 daemonset 创建的 pod 已不再使用直接指定 .spec.nodeName的方式绕过调度器进行调度，而是走默认调度器通过 `nodeAffinity` 的方式调度到每一个节点上。

# 4. 参考资料

- [DaemonSet概念](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/)
- [DaemonSet 滚动更新](https://kubernetes.io/zh/docs/tasks/manage-daemon/update-daemon-set/)
- [daemonset controller 源码分析](https://cloud.tencent.com/developer/article/1555709)