---
author: Yuan Hao
date: 2020-10-12
title: k8s 应用滚动更新
tag: [rolling update, deployment]
---

# k8s 应用滚动更新

## 1. 概念

滚动更新，通常出现在软件或者是系统中。滚动更新与传统更新的不同之处在于：
滚动更新不但提供了更新服务，而且通常还提供了滚动**进度查询**，滚动**历史记录**，
以及最重要的**回滚**等能力。通俗地说，就是具有系统或是软件的主动降级的能力。

## 2. Deployment 滚动更新

Deployment 更新方式有 2 种：
- RollingUpdate
- Recreate

其中，滚动更新是最常见的，阅读代码 `pkg/controller/deployment/deployment_controller.go:648`，
可以看到 2 种方式分别对应的业务逻辑：
```go
func (dc *DeploymentController) syncDeployment(key string) error {
    ...
	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	...
}
```

根据 `d.Spec.Strategy.Type`，若更新策略为 `RollingUpdate`，
则执行 `dc.rolloutRecreate()` 方法，具体逻辑如下：
```go
func (dc *DeploymentController) rolloutRolling(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
    // 1、获取所有的 rs，若没有 newRS 则创建
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)

	// 2、newRS 执行 scale up 操作
	scaledUp, err := dc.reconcileNewReplicaSet(allRSs, newRS, d)
	if err != nil {
		return err
	}
	if scaledUp {
		// Update DeploymentStatus
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// 3、oldRS 执行 scale down 操作
	scaledDown, err := dc.reconcileOldReplicaSets(allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
	if err != nil {
		return err
	}
	if scaledDown {
		// Update DeploymentStatus
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

    // 4、清理过期的 rs
	if deploymentutil.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	// 5、同步 deployment 状态
	return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

### 2.1 滚动更新概述

上面代码中 5 个重要的步骤总结如下：
1. 调用 `getAllReplicaSetsAndSyncRevision()` 获取所有的 rs，若没有 newRS 则创建；
2. 调用 `reconcileNewReplicaSet()` 判断是否需要对 newRS 进行 scaleUp 操作；如果需要 scaleUp，更新 Deployment 的 status，
添加相关的 condition，该 condition 的 type 是 Progressing，表明该 deployment 正在更新中，然后直接返回；
3. 调用 `reconcileOldReplicaSets()` 判断是否需要为 oldRS 进行 scaleDown 操作；如果需要 scaleDown，
把 oldRS 关联的 pod 删掉 maxScaledDown 个，然后更新 Deployment 的 status，添加相关的 condition，直接返回。
这样一来就保证了在滚动更新过程中，新老版本的 Pod 都存在；
4. 如果两者都不是则滚动升级很可能已经完成，此时需要检查 `deployment.Status` 是否已经达到期望状态，
并且根据 `deployment.Spec.RevisionHistoryLimit` 的值清理 oldRSs；
5. 最后，同步 deployment 的状态，使其与期望一致；

从上面的步骤可以看出，滚动更新的过程主要分成一下三个阶段：

{{< mermaid >}}
graph LR
    start((开始)) --> condtion1{newRS need scale up ?}
    condtion1 -- No --> condtion2{oldRS need scale down ?}
    condtion2 -- NO --> x3(3. sync deploment status)
    condtion1 -- YES --> x1(1. newRS scale up)
    x1 --> stop((结束))
    condtion2 -- YES --> x2(2. oldRS scale down)
    x2 --> stop
    x3 --> stop
{{< /mermaid >}}

#### 2.1.1 newRS scale up

阅读代码 `pkg/controller/deployment/rolling.go:68`，详细如下： 
```go
func (dc *DeploymentController) reconcileNewReplicaSet(allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
    // 1、判断副本数是否已达到了期望值
	if *(newRS.Spec.Replicas) == *(deployment.Spec.Replicas) {
		// Scaling not required.
		return false, nil
	}
	
	// 2、判断是否需要 scale down 操作
	if *(newRS.Spec.Replicas) > *(deployment.Spec.Replicas) {
		// Scale down.
		scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, *(deployment.Spec.Replicas), deployment)
		return scaled, err
	}
	
	// 3、计算 newRS 所需要的副本数
	newReplicasCount, err := deploymentutil.NewRSNewReplicas(deployment, allRSs, newRS)
	if err != nil {
		return false, err
	}
	
	// 4、如果需要 scale ，则更新 rs 的 annotation 以及 rs.Spec.Replicas
	scaled, _, err := dc.scaleReplicaSetAndRecordEvent(newRS, newReplicasCount, deployment)
	return scaled, err
}
```
从上面的源码可以得出，`reconcileNewReplicaSet()` 的主要逻辑如下：
1. 判断 `newRS.Spec.Replicas` 和 `deployment.Spec.Replicas` 是否相等，
   如果相等则直接返回，说明已经达到期望状态；
2. 若 `newRS.Spec.Replicas > deployment.Spec.Replicas`，
   则说明 newRS 副本数已经超过期望值，调用 `dc.scaleReplicaSetAndRecordEvent()` 进行 scale down；
1. 此时 `newRS.Spec.Replicas <  deployment.Spec.Replicas`，
调用 `deploymentutil.NewRSNewReplicas()` 为 newRS 计算所需要的副本数，
计算原则遵守 maxSurge 和 maxUnavailable 的约束；
4. 调用 `dc.scaleReplicaSetAndRecordEvent()` 更新 newRS 对象，设置
  `rs.Spec.Replicas`、`rs.Annotations[DesiredReplicasAnnotation]` 
  以及 `rs.Annotations[MaxReplicasAnnotation]`；

其中，计算 newRS 的副本数，是滚动更新核心过程的第一步，
阅读源码 `pkg/controller/deployment/util/deployment_util.go:816`：
```go
func NewRSNewReplicas(deployment *apps.Deployment, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet) (int32, error) {
	switch deployment.Spec.Strategy.Type {
	case apps.RollingUpdateDeploymentStrategyType:
		// 1、计算 maxSurge 值，向上取整
		maxSurge, err := intstrutil.GetValueFromIntOrPercent(deployment.Spec.Strategy.RollingUpdate.MaxSurge, int(*(deployment.Spec.Replicas)), true)
		if err != nil {
			return 0, err
		}
		
		// 2、累加 rs.Spec.Replicas 获取 currentPodCount
		currentPodCount := GetReplicaCountForReplicaSets(allRSs)
		maxTotalPods := *(deployment.Spec.Replicas) + int32(maxSurge)
		if currentPodCount >= maxTotalPods {
			// Cannot scale up.
			return *(newRS.Spec.Replicas), nil
		}
		
		// 3、计算 scaleUpCount，结果不超过期望值
		scaleUpCount := maxTotalPods - currentPodCount
		scaleUpCount = int32(integer.IntMin(int(scaleUpCount), int(*(deployment.Spec.Replicas)-*(newRS.Spec.Replicas))))
		
		return *(newRS.Spec.Replicas) + scaleUpCount, nil
	case apps.RecreateDeploymentStrategyType:
		return *(deployment.Spec.Replicas), nil
	default:
		return 0, fmt.Errorf("deployment type %v isn't supported", deployment.Spec.Strategy.Type)
	}
}
```

可知 `NewRSNewReplicas()` 的主要逻辑如下：
1. 判断更新策略；
2. 计算 maxSurge 值；
3. 通过 allRSs 计算 currentPodCount 的值；
4. 最后计算 scaleUpCount 值；

#### 2.1.2 oldRS scale down

同理，oldRS 规模缩小，阅读源码 `pkg/controller/deployment/rolling.go:68`:
```go
func (dc *DeploymentController) reconcileOldReplicaSets(allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
    // 1、计算 oldPodsCount
	oldPodsCount := deploymentutil.GetReplicaCountForReplicaSets(oldRSs)
	if oldPodsCount == 0 {
		// Can't scale down further
		return false, nil
	}

    // 2、计算 maxUnavailable
	allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
	klog.V(4).Infof("New replica set %s/%s has %d available pods.", newRS.Namespace, newRS.Name, newRS.Status.AvailableReplicas)
	maxUnavailable := deploymentutil.MaxUnavailable(*deployment)

	
	// 3、计算 maxScaledDown
	minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
	newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
	maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
	if maxScaledDown <= 0 {
		return false, nil
	}

	// 4、清理异常的 rs
	oldRSs, cleanupCount, err := dc.cleanupUnhealthyReplicas(oldRSs, deployment, maxScaledDown)
	if err != nil {
		return false, nil
	}
	klog.V(4).Infof("Cleaned up unhealthy replicas from old RSes by %d", cleanupCount)

	// 5、缩容 old rs
	allRSs = append(oldRSs, newRS)
	scaledDownCount, err := dc.scaleDownOldReplicaSetsForRollingUpdate(allRSs, oldRSs, deployment)
	if err != nil {
		return false, nil
	}
	klog.V(4).Infof("Scaled down old RSes of deployment %s by %d", deployment.Name, scaledDownCount)

	totalScaledDown := cleanupCount + scaledDownCount
	return totalScaledDown > 0, nil
}
```

通过上面的代码可知，`reconcileOldReplicaSets()`  的主要逻辑如下：
1. 通过 oldRSs 和 allRSs 获取 oldPodsCount 和 allPodsCount；
2. 计算 deployment 的 maxUnavailable、minAvailable、newRSUnavailablePodCount、maxScaledDown 值，
当 deployment 的 maxSurge 和 maxUnavailable 值为百分数时，
计算 maxSurge 向上取整而 maxUnavailable 则向下取整；
3. 清理异常的 rs；
4. 计算 oldRS 的 scaleDownCount；
5. 最后 oldRS 缩容；

### 2.2 滚动更新总结

通过上面的代码可以看出，滚动更新过程中主要是通过调用 `reconcileNewReplicaSet()` 对 newRS 不断扩容，
调用 `reconcileOldReplicaSets()` 对 oldRS 不断缩容，最终达到期望状态，并且在整个升级过程中，
都严格遵守 maxSurge 和 maxUnavailable 的约束。

不论是在 scale up 或者 scale down 中都是调用 `scaleReplicaSetAndRecordEvent()` 执行，
而 `scaleReplicaSetAndRecordEvent()` 又会调用 `scaleReplicaSet()`，
扩缩容都是更新 `rs.Annotations` 以及 `rs.Spec.Replicas`。

整体流程如下图所示：

{{< mermaid >}}
graph LR
    op1(newRS scale up) --> op3[dc.scaleReplicaSetAndRecordEvent]
    op2(oldRS scale down) --> op3(dc.scaleReplicaSetAndRecordEvent)
    op3(dc.scaleReplicaSetAndRecordEvent) --> op4(dc.scaleReplicaSet)
{{< /mermaid >}}

### 2.3 滚动更新示例

1. 创建一个deployment，replica = 10
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 10
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
        image: nginx:1.18.0
        ports:
        - containerPort: 80
```

10 个 Pod 创建成功后如下所示：
```shell script
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-67dfd6c8f9   10        10        10      70s
```

2. 更新 nginx-deployment 的镜像，默认使用滚动更新的方式
```shell script
$ kubectl set image deploy/nginx-deployment nginx-deployment=nginx:1.19.1
```

此时通过源码可知会计算该 deployment 的 maxSurge=3，maxUnavailable=2，maxAvailable=13，计算方法如下所示：
```go
// 向上取整 maxSurge = 10 * 0.25 = 3
maxSurge = replicas * deployment.spec.strategy.rollingUpdate.maxSurge
// 向下取整 maxUnavailable = 10 * 0.25 = 2
maxUnavailable = replicas * deployment.spec.strategy.rollingUpdate.maxUnavailable
// maxAvailable = 10 + 3 = 13
maxAvailable = replicas + MaxSurge
```

如上面代码所说，更新时首先创建 newRS，然后为其设定 replicas，计算 newRS 的 replicas 值的方法在 `NewRSNewReplicas()` 中，
此时计算出 replicas 结果为 3，然后更新 deployment 的 annotation，创建 events，本次 syncLoop 完成。
等到下一个 syncLoop 时，所有 rs 的 replicas 已经达到最大值 10 + 3 = 13，此时需要 oldRS 缩容。
scale down 的数量是通过以下公式得到的：
```go
// 13 = 10 + 3
allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
// 8 = 10 - 2
minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
// ???
newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
// 13 - 8 - ???
maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
```

allPodsCount = 13，minAvailable = 8 ，newRSUnavailablePodCount 此时不确定，但是值在 [0,3] 范围内。
此时假设 newRS 的 3 个 pod 还处于 containerCreating 状态，则 newRSUnavailablePodCount = 3，
根据以上公式计算所知 maxScaledDown = 2，则 oldRS 需要缩容 2 个 pod，其 replicas 需要改为 8，此时该 syncLoop 完成。
下一个 syncLoop 时在 scaleUp 处计算得知 scaleUpCount = 13 - 8 - 3 = 2， 
此时 newRS 需要更新 replicase 增加 2。以此轮询直到 newRS 扩容到 10，oldRS 缩容至 0。

对于上面的示例，可以使用 kubectl get rs -w 进行观察，以下为输出：
```shell script
$ kubectl get rs -w
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5bbdfb5879   10        10        5       3s
nginx-deployment-67dfd6c8f9   3         3         3       4m47s
nginx-deployment-5bbdfb5879   10        10        6       3s
nginx-deployment-67dfd6c8f9   2         3         3       4m47s
nginx-deployment-67dfd6c8f9   2         3         3       4m47s
nginx-deployment-67dfd6c8f9   2         2         2       4m47s
nginx-deployment-5bbdfb5879   10        10        7       4s
nginx-deployment-67dfd6c8f9   1         2         2       4m48s
nginx-deployment-67dfd6c8f9   1         2         2       4m48s
nginx-deployment-67dfd6c8f9   1         1         1       4m48s
nginx-deployment-5bbdfb5879   10        10        8       4s
nginx-deployment-67dfd6c8f9   0         1         1       4m48s
nginx-deployment-67dfd6c8f9   0         1         1       4m48s
nginx-deployment-67dfd6c8f9   0         0         0       4m48s
nginx-deployment-5bbdfb5879   10        10        9       5s
nginx-deployment-5bbdfb5879   10        10        10      6s
```