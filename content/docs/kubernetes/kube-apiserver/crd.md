---
author: Yuan Hao
date: 2019-10-14
title: CRD 入门和使用
tag: [crd]
---

# 1. Customer Resource

自定义资源是 Kubernetes API 的扩展，本文将讨什么时候应该向 Kubernetes 集群添加自定义资源以及何时使用独立服务。它描述了添加自定义资源的两种方法以及如何在它们之间进行选择。

## 1.1 自定义资源概述

资源就是 Kubernetes API 集合中的某个的对象。例如，内置 pods 资源包含 Pod 对象的集合。

自定义资源是扩展了 Kubernetes API，但在默认的 API 集合中是不可用的，因为没有对应的 controller 处理业务逻辑。使用自定义资源，不仅可以解耦大多数的自研功能，还可用使得 Kubernetes 更加模块化。

自定义资源可以通过动态注册的方法，于正在 running 的集群中创建、删除和更新，并且与集群本身的资源是相互独立的。当自定义资源被创建，就可以通过 kubectl 创建和访问对象，和对内置资源的操作完全一致。

## 1.2 是否需要自定义资源

在创建新的 API 时，应该考虑与 Kubernetes 的 API 聚合，还是让 API 独立运行。

| API 聚合                      | API 独立           |
| ---------------------------- | ----------------- |
| 声明式                       | 非声明式          |
| kubectl 可读可写              | 不需要 kubectl 支持 |
| 接受 k8s 的 REST 限制            | 特定 API 路径       |
| 可限定为 cluster 或者 namespace | 不适用 namespace   |
| 重用 k8s API 支持的功能        | 不需要这些功能    |

## 1.3 声明式 API VS 命令式 API

在声明式 API，通过具有以下特性：

- 对象颗粒度小
- 基础结构的定义
- 读写多，更新少，操作主要为 CRUD
- API 表期望状态

命令式 API，通过具有以下特性：

- 同步响应
- RPC 调用
- 大量数据存储（大对象或多对象）
- 高带宽访问
- 操作非 CRUD
- API 不易抽象

# 2. Customer Resource Definition

Kubernetes 提供了两种向集群添加自定义资源的方法：

- 创建自定义 API server 并聚合到 API 中
- CRDs

## 2.1 Customer Resource Definition 介绍

> kubernetes: v1.19.0

通过 CRD 创建自定义资源，只要符合 CRD 结构体定义，均可以被 Kubernetes 所接收，CRD 的定义和 k8s 原生资源的定义非常类似，都是由 4 个子资源组成：TypeMeta、ObjectMeta、Spec、Status，具体代码如下：

`staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/types.go:354`
```go
type CustomResourceDefinition struct {
	metav1.TypeMeta
	metav1.ObjectMeta
	Spec CustomResourceDefinitionSpec 
	Status CustomResourceDefinitionStatus
}
```

下面主要来看下 Spec 和 Status 字段：

```go
type CustomResourceDefinitionSpec struct {
	Group string
	// 定义复数、单数，简称，Kind，ListKind，Categories
	Names CustomResourceDefinitionNames
	// 表示集群级别或 namespace 级别
	Scope ResourceScope
	// 定义 CR 所有支持的版本号
	Versions []CustomResourceDefinitionVersion
	// 定义不同版本之间的转换方法
	Conversion *CustomResourceConversion
	// 是否保留未知字段，apiVersion、kind、metadata 等总是保留
	PreserveUnknownFields *bool
}

type CustomResourceDefinitionStatus struct {
	// 描述资源的当前状态，即 status.conditions
	Conditions []CustomResourceDefinitionCondition
	// 与 spec.names 类似，即 status.acceptNames
	AcceptedNames CustomResourceDefinitionNames
	// 资源曾经存在的版本，可在 etcd 中存储，供迁移用
	StoredVersions []string
}

```

## 2.2 CRD 定义示例

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名称必须符合下面的格式：<plural>.<group>
  name: foos.samplecontroller.io
spec:
  # REST API 使用的组名称：/apis/<group>/<version>
  group: samplecontroller.io
  # REST API 使用的版本号：/apis/<group>/<version>
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        # 校验方法
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                #类型校验
                replicas:
                  type: integer
                deploymentName:
                  type: string
  # Namespaced 或 Cluster
  scope: Namespaced
  names:
    # URL 中使用的复数名称：/apis/<group>/<version>/<plural>
    plural: foos
    # CLI 中使用的单数名称
    singular: foo
    # CamelCased 格式的单数类型。在清单文件中使用
    kind: Foo
    # 资源的 List 类型，默认是 “`kind`List”
    listKind: FooList
    # CLI 中使用的资源简称
    shortNames:
      - fo
    # 分组
    categories:
      - all
```

## 2.3 Customer Controller

只定义资源，不实现资源控制器是没有任何意义的。自定义控制器能够完成业务逻辑，最主要是依赖 client-go 库的各个组件的交互。下图展示它们之间的关系：

![client-go-controller-interaction.jpeg](/kubernetes/kube-apiserver/crd/client-go-controller-interaction.jpeg)

通过图示，可以看到几个核心组件的交互流程，蓝色表示 client-go，黄色是自定义 controller，各组件作用介绍如下：

### 2.3.1 client-go 组件

- **Reflector**：reflector 用来 watch 特定的 k8s API 资源。具体的实现是通过 ListAndWatch 的方法，watch 可以是 k8s 内建的资源或者是自定义的资源。当 reflector 通过 watch API 接收到有关新资源实例存在的通知时，它使用相应的列表 API 获取新创建的对象，并将其放入 watchHandler 函数内的 Delta FIFO 队列中。
- **Informer**：informer 从 Delta FIFO 队列中弹出对象。执行此操作的功能是 processLoop。base controller 的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。
- **Indexer**：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer 可以根据多个索引函数维护索引。Indexer 使用线程安全的数据存储来存储对象及其键。 在 Store 中定义了一个名为 MetaNamespaceKeyFunc 的默认函数，该函数生成对象的键作为该对象的 namespace/name 组合。

### 2.3.2 自定义 controller 组件

- **Informer reference**：指的是 Informer 实例的引用，定义如何使用自定义资源对象。自定义控制器代码需要创建对应的 Informer。
- **Indexer reference**：自定义控制器对 Indexer 实例的引用。自定义控制器需要创建对应的 Indexer。
- **Resource Event Handlers**：资源事件回调函数，当它想要将对象传递给控制器时，它将被调用。编写这些函数的典型模式是获取调度对象的 key，并将该 key 排入工作队列以进行进一步处理。
- **Work queue**：任务队列。编写资源事件处理程序函数以提取传递的对象的 key 并将其添加到任务队列。
- **Process Item**：处理任务队列中对象的函数，这些函数通常使用 Indexer 引用或 Listing 包装器来重试与该 key 对应的对象。

简单的说，整个处理流程大概为：Reflector 通过检测 Kubernetes API 来跟踪该扩展资源类型的变化，一旦发现有变化，就将该 Object 存储队列中，Informer 循环取出该 Object 并将其存入 Indexer 进行检索，同时触发 Callback 回调函数，并将变更的 Object Key 信息放入到工作队列中，此时自定义 Controller 里面的 Process Item 就会获取工作队列里面的 Key，并从 Indexer 中获取 Key 对应的 Object，从而进行相关的业务处理。

# 3. sample-controller 使用

## 3.1 资源准备

### 3.1.1 foo.crd.yaml

使用 2.2 节的示例。

### 3.1.2 example.foo.yaml

创建 foo 资源类型的对象：

```yaml
apiVersion: samplecontroller.io/v1alpha1
kind: Foo
metadata:
  name: example-foo
spec:
  deploymentName: example-foo
  replicas: 1
```

## 3.2 sample-controller 部署

### 3.2.1 代码

获取示例代码：

```s
git clone https://github.com/kubernetes/sample-controller.git
cd sample-controller
```

### 3.2.2 编译

在编译二进制之前，要修改 `pkg/apis/samplecontroller/register.go:21` GroupName 的定义，k8s.io 和 kubernetes.io 后缀都是官方保留字段，不建议 CRD 使用。所以将其修改为 `samplecontroller.io`。

```s
go build -o sample-controller .
```

### 3.2.3 启动

使用节点上的 kubeconfig 文件启动 controller：
```s
./sample-controller -kubeconfig=$HOME/.kube/config
```

启动时，会提示 watch 不到资源，因为还没创建定义：
```s
I1026 23:40:56.529233   57978 controller.go:115] Setting up event handlers
I1026 23:40:56.529444   57978 controller.go:156] Starting Foo controller
I1026 23:40:56.529450   57978 controller.go:159] Waiting for informer caches to sync
E1026 23:40:56.554564   57978 reflector.go:138] k8s.io/sample-controller/pkg/generated/informers/externalversions/factory.go:117: Failed to watch *v1alpha1.Foo: failed to list *v1alpha1.Foo: the server could not find the requested resource (get foos.samplecontroller.io)
E1026 23:40:57.841194   57978 reflector.go:138] k8s.io/sample-controller/pkg/generated/informers/externalversions/factory.go:117: Failed to watch *v1alpha1.Foo: failed to list *v1alpha1.Foo: the server could not find the requested resource (get foos.samplecontroller.io)
```

### 3.2.4 创建资源

```s
$ kubectl create  -f foos.crd.yaml
customresourcedefinition.apiextensions.k8s.io/foos.samplecontroller.io created

$ kubectl create  -f example.foos.yaml
foo.samplecontroller.io/example-foo created
```
创建完成，可以使用 `kubectl get crd` 查看资源定义和对象：

```s
$ kubectl get crd
NAME                       CREATED AT
clusterversions.cfe.io     2020-10-26T14:43:27Z
foos.samplecontroller.io   2020-10-26T15:41:52Z

$ kubectl get foos
NAME          AGE
example-foo   19m
```

sample-controller 会 watch 到新对象 default/example-foo：

```s
I1026 23:42:03.232577   57978 controller.go:164] Starting workers
I1026 23:42:03.232607   57978 controller.go:170] Started workers
I1026 23:42:03.515970   57978 controller.go:228] Successfully synced 'default/example-foo'
I1026 23:42:03.516099   57978 event.go:291] "Event occurred" object="default/example-foo" kind="Foo" apiVersion="samplecontroller.io/v1alpha1" type="Normal" reason="Synced" message="Foo synced successfully"
```

### 3.2.5 校验

检查 controller 创建 Deployment 对象：

```s
$ kubectl get deploy                       
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
example-foo   1/1     1            1           3m1s

$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
example-foo-54dc4db9fc-4r8pp   1/1     Running   0          3m8s
shell-demo                     1/1     Running   9          43d
web-0                          1/1     Running   16         78d
web-1                          1/1     Running   16         78d
```

# 4. 参考资料
- [定制资源](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [controller-client-go](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)
- [sample-controller](https://github.com/kubernetes/sample-controller/blob/master/README.md)