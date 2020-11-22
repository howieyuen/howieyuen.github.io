---
author: Yuan Hao
date: 2020-10-14
title: 垃圾回收
tag: [garbage collector]
---

# 1. 序言 

## 1.1 什么是垃圾回收

参考 Java 中的概念，垃圾回收（Garbage Collection）是 JVM 垃圾回收器提供的一种用于在空闲时间不定时回收无任何对象引用的对象占据的内存空间的一种机制。
垃圾回收回收的是无任何引用的对象占据的内存空间而不是对象本身。换言之，垃圾回收只会负责释放那些对象占有的内存。
对象是个抽象的词，包括引用和其占据的内存空间。当对象没有任何引用时其占据的内存空间随即被收回备用，此时对象也就被销毁。

因此，垃圾回收关注的是**无任何引用的对象**。在 kubernetes 中，对象的引用关系又是怎样的呢？

## 1.2 k8s 中的对象引用

某些 kubernetes 对象是其他一些对象的属主。例如一个 ReplicaSet 是一组 Pod 的属主；反之这组 Pod 就是此 ReplicaSet 的附属。
每个附属对象具有一个指向属主对象的 `metadata.ownerReference` 字段。

Kubernetes 会自动为 ReplicationController、ReplicaSet、StatefulSet、DaemonSet、Deployment、Job 和 CronJob 自动设置 `ownerReference` 的值。
也可以通过手动设置 `ownerReference` 的值，来指定属主和附属之间的关系。

先看一个 Pod 的详细信息，例如下面的配置显示 Pod 的属主是名为 my-replicaset 的 ReplicaSet： 
```yaml
apiVersion: v1
kind: Pod
metadata:
  ...
  ownerReferences:
  - apiVersion: apps/v1
    controller: true
    blockOwnerDeletion: true
    kind: ReplicaSet
    name: my-rs
    uid: d9607e19-f88f-11e6-a518-42010a800195
  ...
```

下面的源码是 `ownerReference` 的结构：
```go
type OwnerReference struct {
	
	APIVersion string `json:"apiVersion" protobuf:"bytes,5,opt,name=apiVersion"`
	
	Kind string `json:"kind" protobuf:"bytes,1,opt,name=kind"`
	
	Name string `json:"name" protobuf:"bytes,3,opt,name=name"`
	
	UID types.UID `json:"uid" protobuf:"bytes,4,opt,name=uid,casttype=k8s.io/apimachinery/pkg/types.UID"`
	
	Controller *bool `json:"controller,omitempty" protobuf:"varint,6,opt,name=controller"`
	
	BlockOwnerDeletion *bool `json:"blockOwnerDeletion,omitempty" protobuf:"varint,7,opt,name=blockOwnerDeletion"`
}
```
上面的结构中没有`Namespace` 属性，这是为什么呢？

> 根据设计，kubernetes 不允许跨命名空间指定属主。也就是说： 
> 1. 有 `Namespace` 属性的附属，只能指定同一 `Namespace` 中的或者集群范围的属主。 
> 2. `Cluster` 级别的附属，只能指定集群范围的属主，不能指定命名空间范围的属主。

如此一来，有 `Namespace` 属性的对象，它的属主要么是与自己是在同一个 `Namespace` 下，可以复用；要么是集群级别的对象，没有 `Namespace` 属性。
而 `Cluster` 级别的对象，它的属主也必须是 `Cluster` 级别，没有 `Namespace` 属性。
因此，`OwnerReference` 结构中不需要 `Namespace` 属性。

上面 `OwnerReference` 的结构体中，最后一个字段 `BlockOwnerDeletion` 字面意思就是阻止属主删除，那么 k8s 在删除对象上与垃圾收集有什么关系？
垃圾收集具体是如何执行的呢？后文继续分析。

# 2. k8s 中的垃圾回收

删除对象时，可以指定该对象的附属是否也自动删除。Kubernetes 中有三种删除模式：
- 级联删除
    - Foreground 模式
    - Background 模式
- 非级联删除
    - Orphan 模式 

在 kubernetes v1.9 版本之前，大部分 controller 的默认删除策略为 `Orphan`，从 v1.9 开始，对 apps/v1 下的资源默认使用 `Background` 模式。

针对 Pod 存在特有的删除方式：`Gracefully terminate`，允许优雅地终止容器。
先发送 TERM 信号，过了宽限期还未终止，则发送 KILL 信号。
kube-apiserver 先设置 ObjectMeta.DeletionGracePeriodSeconds，默认为 30s，
再由 kubelet 发送删除请求，请求参数中 DeleteOptions.GracePeriodSeconds = 0，
kube-apiserver 判断到 lastGraceful = options.GracePeriodSeconds = 0，就直接删除对象了。

## 2.1 Foreground 模式

在 Foreground 模式下，待删除对象首先进入 `deletion in progress` 状态。 在此状态下存在如下的场景：
- 对象仍然可以通过 REST API 获取。
- 会设置对象的 `deletionTimestamp` 字段。
- 对象的 `metadata.finalizers` 字段包含值 `foregroundDeletion`。

只有对象被设置为 `deletion in progress` 状态时，垃圾收集器才会删除对象的所有附属。 
垃圾收集器在删除了所有有阻塞能力的附属（对象的 `ownerReference.blockOwnerDeletion=true`） 之后，再删除属主对象。

> 注意，在 Foreground 模式下，只有设置了 `ownerReference.blockOwnerDeletion=true` 的附属才能阻止属主对象被删除。 
在 Kubernetes 1.7 版本增加了准入控制器，基于属主对象上的删除权限来限制用户设置 `ownerReference.blockOwnerDeletion=true`，
这样未经授权的附属不能够阻止属主对象的删除。

如果一个对象的 ownerReference 字段被一个控制器（例如 Deployment 或 ReplicaSet）设置， blockOwnerDeletion 也会被自动设置，不需要手动修改。

## 2.2 Background 模式

在 Background 模式下，Kubernetes 会立即删除属主对象，之后垃圾收集器会在后台删除其附属对象。

## 2.3 Orphan 模式

与 Foreground 模式类似，待删除对象首先进入 `deletion in progress` 状态。 在此状态下存在如下的场景：
- 对象仍然可以通过 REST API 获取。
- 会设置对象的 `deletionTimestamp` 字段。
- 对象的 `metadata.finalizers` 字段包含值 `orphan`。

与 Foreground 模式不同的是，不会自动删除它的依赖或者是子对象，这些残留的依赖被称作是原对象的**孤儿对象**。

例如执行以下命令时，会使用 `Orphan` 策略进行删除，此时 ds 的依赖对象 `controllerrevision` 不会被删除：

```s
kubectl delete rs my-rs --cascade=false
```
或者使用 curl 命令：
```s
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-rs \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
  -H "Content-Type: application/json"
```

## 2.4 finalizer 机制

finalizer 是在对象删除之前需要执行的逻辑。每当 finalizer 成功运行之后，就会将它自己从 `Finalizers` 数组中删除，
当最后一个 finalizer 被删除之后，API Server 就会删除该对象。finalizer 提供了一个通用的 API，
它的功能不只是用于阻止级联删除，还能过通过它在对象删除之前加入钩子。

```go
type ObjectMeta struct {
	// ...
	Finalizers []string
}
```

k8s 中默认有两种 finalizer：`OrphanFinalizer` 和 `ForegroundFinalizer`，分别对应 `Orphan` 模式和 `Foreground` 模式。
当然，finalizer 不仅仅支持以上两种字段，在使用自定义 controller 时，也可以在 CustomResource 中设置自定义的 finalizer 标识。

在默认情况下，删除一个对象会删除它的全部依赖，但是在一些特定情况下，我们只是想删除当前对象本身并不想造成复杂地级联删除，
如此 `OrphanFinalizer` 应运而生。`OrphanFinalizer` 会监听对象的更新事件并将它自己从它全部依赖对象的 `OwnerReferences` 数组中删除，
与此同时会删除所有依赖对象中已经失效的 `OwnerReferences` 并将 `OrphanFinalizer` 从 `Finalizers` 数组中删除。

通过 `OrphanFinalizer` 我们能够在删除一个 Kubernetes 对象时保留它的全部依赖，为使用者提供一种更灵活的方法来保留和删除对象。

# 3. 参考资料

- [垃圾收集](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection)
- [garbage collector controller 源码分析](https://cloud.tencent.com/developer/article/1562130)
- [Kubernetes API 资源对象的删除和 GarbageCollector Controller](http://yangxikun.github.io/kubernetes/2020/03/17/kubernetes-delete-obj-and-garbage-collector-controller.html)