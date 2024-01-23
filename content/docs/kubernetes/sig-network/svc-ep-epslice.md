---
author: howieyuen
date: 2023-01-31
title: Service 与 Endpoints、EndpointSlice 那些事
tag: [Service, Endpoints, EndpointSlice]
categories: [Kubernetes]
---

# Service 与 Endpoints、EndpointSlice 那些事

## Service 资源

k8s Service 定义了这样一种抽象：逻辑上的一组 Pod，一种可以访问它们的策略 —— 通常称为微服务。
Service 所针对的 Pod 集合通常是通过 LabelSelector 来确定的。
LabelSelector 选中的 Pod，将会自动创建与 Service 同名且同 Namespace 的 Endpoint 对象，将 Pod 的 IP 写入其中。

如果没有 LabelSelector，用户可能是希望 Service 关联的后端是集群外的服务，或者是其他 Namespace；
如果用户还想要使用负载均衡服务，需要自行创建与 Service 同名的 Endpoints 对象。

## Service 类型

k8s Service 分为 4 种类型（`svc.spec.type`）：
- ClusterIP：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。
  这也是默认的 `ServiceType`。
- NodePort：通过每个节点上的 IP 和静态端口（NodePort）暴露服务。
  NodePort 服务会路由到自动创建的 ClusterIP 服务。
  通过请求 `<node ip>:<node port>`，你可以从集群的外部访问一个 `NodePort` 服务。
- LoadBalance：使用云提供商的负载均衡器向外部暴露服务。
  外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上。
- ExternalName：通过返回 `CNAME` 和对应值，可以将服务映射到 `externalName` 字段的内容；
  例如，`foo.bar.example.com`，无需创建任何类型代理。

## Headless Service

无头服务，也可以算是 Service 的一种类型，有时不需要或不想要负载均衡，以及单独的 Service IP，
可以通过指定 Cluster IP（`spec.clusterIP`）的值为 None 来创建 Headless Service。

如果有 LabelSelector，Endpoints Controller 在 API 中创建 Endpoints 对象，
并且修改 DNS 配置返回 IP 地址，通过这个地址直接到达 Service 的后端 Pod 上。

如果每有 LabelSelector，Endpoints Controller 不会创建 Endpoints 对象。
但DNS 系统会查找和配置，无论是：
- 对于 ExternalName 类型的服务，查找其 CNAME 记录
- 对所有其他类型的服务，查找与 Service 名称相同的任何 Endpoints 的记录

## Endpoints

刚才提到，用户一般不需要主动创建 Endpoints 记录，Service 会根据其类型，决定是否创建，
因此 Service 和 Endpoints 之间**不是属主和附属的关系**（`ownerReference`）。

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  creationTimestamp: "2022-09-08T09:55:14Z"
  name: frontend
  namespace: sampleapp
  resourceVersion: "1655960"
  uid: 4a8381ac-a4b6-477e-a1ed-ddf3d9f47d2c
subsets:
- addresses:
  - ip: 172.17.0.21
    nodeName: minikube
    targetRef:
      kind: Pod
      name: sampleappprod-8445c57bd7-c7vnt
      namespace: sampleapp
      resourceVersion: "1655943"
      uid: 38f10fd0-a30c-4ede-87e5-e446d571f054
  - ip: 172.17.0.25
    nodeName: minikube
    targetRef:
      kind: Pod
      name: sampleappprod-8445c57bd7-4bpdd
      namespace: sampleapp
      resourceVersion: "1655956"
      uid: 3c760d52-d389-4fb2-8219-d50fc418e831
  ports:
  - port: 80
    protocol: TCP
```

## EndpointSlice

### 动机

由于任一 Service 的所有网络端点都保存在同一个 Endpoints 资源中，
这类资源可能变得非常巨大，而这一变化会影响到 Kubernetes 组件的性能，
并在 Endpoints 变化时产生大量的网络流量和额外的处理。
EndpointSlice 能够缓解这一问题，还能为一些诸如拓扑路由这类的额外功能提供一个可扩展的平台。

### 定义

在 Kubernetes 中，EndpointSlice 包含对一组网络端点的引用。
控制面会自动为设置了 LabelSelector 的 Kubernetes Service 创建 EndpointSlice。
这些 EndpointSlice 将包含对与 Service 的 LabelSelector 匹配的所有 Pod 的引用。
EndpointSlice 通过唯一的协议、端口号和 Service 名称将 Endpoints 组织在一起。

### 属主

在大多数场合下，EndpointSlice 都由某个 Service 所有，
因为 EndpointSlice 正是为该服务跟踪记录其 Endpoints。
这一属主关系是通过为每个 EndpointSlice 设置 ownerReference，
同时设置 kubernetes.io/service-name 标签来标明的，
目的是方便查找属于某 Service 的所有 EndpointSlice。

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  creationTimestamp: "2022-09-08T09:55:14Z"
  generateName: frontend-
  generation: 11
  labels:
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
    kubernetes.io/service-name: frontend
  name: frontend-xq96c
  namespace: sampleapp
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: frontend
    uid: 2e6d626f-e8aa-4edb-a42f-19caf7acdf46
  resourceVersion: "1655961"
  uid: 42c13581-9ba7-4d7e-95af-57742c028446
addressType: IPv4
endpoints:
- addresses:
  - 172.17.0.25
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: minikube
  targetRef:
    kind: Pod
    name: sampleappprod-8445c57bd7-4bpdd
    namespace: sampleapp
    resourceVersion: "1655956"
    uid: 3c760d52-d389-4fb2-8219-d50fc418e831
- addresses:
  - 172.17.0.21
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: minikube
  targetRef:
    kind: Pod
    name: sampleappprod-8445c57bd7-c7vnt
    namespace: sampleapp
    resourceVersion: "1655943"
    uid: 38f10fd0-a30c-4ede-87e5-e446d571f054
ports:
- name: ""
  port: 80
  protocol: TCP
```

## 关联关系

![](/kubernetes/sig-network/relationship.png)

## 参考资料

- [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
- [https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)