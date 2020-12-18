---
author: Yuan Hao
date: 2020-03-22
title: 高级调度
tag: [scheduling, taint, toleration]
---

# 1. 使用 taint 和 toleration 阻止节点调度到特定节点

## 1.1 taint 和 toleration

taint，是在不修改已有 Pod 的前提下，通过在节点上添加污点信息，拒绝 Pod 的部署。
只有当一个 Pod 容忍某个节点的 taint 时，才能被调度到此节点上。

**显示节点 taint 信息**
```yaml
kubectl describe node master.k8s

...
Name: master.k8s
Role:
Labels:      
    beta.kubernetes.io/arch=amd64
    beta.kubernetes.io/os=linux
    kubernetes.io/hostname=master.k8s
Annotations: 
    node.alpha.kubernetes.io/ttl=0
Taints:     
    node-role.kubernetes.io/master:NoSchedule
...
```
主节点上包含一个污点，污点包含一个 key 和 value，以及一个 effect，格式为<key>=<value>:<effect>。
上面的污点信息表示，key 为 node-role.kubernetes.io/master，value 是空，effect 是 NoSchedule。

**显示 Pod tolerations**
```yaml
kubectl describe Pod kube-proxy-as92 -n kube-system

...
Tolerations: 
    node-role.kubernetes.io/master:=NoSchedule
    node.alpha.kubernetes.io/notReady=:Exists:NoExecute
    node.alpha.kubernetes.io/unreachable=:Exists:NoExecute
...
```
第一个 toleration 匹配了主节点的 taint，表示允许这个 Pod 调度到主节点上。

**了解污点效果**
另外 2 个在 kube-proxy Pod 上容忍定义了当前节点状态是没有 ready 或者是 unreachable 时，
该 Pod 允许运行在该节点上多长时间。污点可以包含以下三种效果：
- NoSchedule，表示如果 pod 没有容忍此污点，将无法调度
- PreferNoSchedule，是上面的宽松版本，如果没有其他节点可调度，依旧可以调度到本节点
- NoExecute，上 2 个在调度期间起作用，而此设定也会影响正在运行中的 Pod。
  如果节点上的 Pod 没有容忍此污点，会被驱逐。

## 1.2 在节点上定义污点
```s
kubectl taint node node1.k8s node-type=production:NoSchedule
```
此命令给节点添加污点，key 为 node-type，value 为 production，effect 为 NoSchedule。
如果现在部署一个常规 Pod 多个副本，没有一个 Pod 会调度到此节点上。

## 1.3 在 Pod 上添加容忍
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod
spec: 
  replicas: 5
  template:
    spec:
    ...
    tolertations:
    - key: node-type
      operator: Equals
      value: production
      effect: NoSchedule
```
部署此 deployment，Pod 即可调度到 node1.k8s 节点上。

## 1.4 了解污点和容忍的使用场景

**调度时使用污点和容忍**

污点可以组织新 Pod 的调度，或者定义非有限调度节点，甚至是将已有 Pod 驱逐。
例如，一个集群分成多个部分，只允许开发团队将 Pod 部署到特定节点上；或者某些特殊 Pod 依赖硬件配置。

**配置节点失效后的 pod 重新调度最长等待时间**

容忍程度也可以配置，当某个 Pod 所在节点 NotReady 或者 Unreachable 是，
k8s 可以等待该 Pod 重新调度之前的最长等待时间。

```yaml
...
tolerations:
- effect: NoExecute
  key: node.alpha.kubernetes.io/notReady
  operator: Exists
  tolerationSeconds: 300
- effect: NoExecute
  key: node.alpha.kubernetes.io/unreachable
  operator: Exists
  tolerationSeconds: 300
...
```
上面 2 个容忍表示，该 Pod 容忍所在节点处于 notReady 和 unreachable 状态维持 300s。超时后，再重新调度。

# 2. 使用节点亲和将 Pod 调度到指定节点

污点可以拒绝调度，亲和允许调度。早期的 k8s 版本，节点亲和，就是 Pod 中的 nodeSelector 字段。
与节点选择器类似，每个节点都可以配置亲和性规则，可以是硬性标注，也可以是偏好。

**检查节点默认标签**

```yaml
kubectl describe node gke-kubia-default-adj12knzf-2dascjb

Name: gke-kubia-default-adj12knzf-2dascjb
Role:
Labels: beta.kubernetes.io/arch=amd64
        beta.kubernetes.io/os=linux
        failure-domain.beta.kubernetes.io/region=europe-west1
        failure-domain.beta.kubernetes.io/zone=europe-west1-d
        kubernetes.io/hostname=gke-kubia-default-adj12knzf-2dascjb
```
这是一个 gke 的节点，最后三个标签涉及到亲和性，分别表示所在地理地域，可用性区域，和主机名。

## 2.1 指定强制节点亲和性规则

下面展示了只能被部署到含有 gpu=true 标签的节点上的 Pod：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec: 
  affinity:
    nodeAffinity:
      requiredDuringSchdulingIgnoredDuringExecution:
        lableSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
```

**较长的节点亲和性属性名的含义**

- requireDuringScheduling... 表示该字段下定义的规则，为了让 Pod 调度到该节点上，必须满足的标签。
- ...IgnoredDuringExecution 表明该字段下的规则，不会影响已经在节点上的 Pod。

**了解节点选择器的条件**

lableSelectorTerms 和 matchExpressions 定义了节点的标签必须满足哪一种表达方式。

## 2.2 调度 Pod 时优先考虑某些节点

节点亲和性和节点选择器相比，最大的好处就是，
当调度某一个 Pod，指定可以优先考某些节点，如果节点均无法满足，调度结果也是可以接受的。
这是由 preferdDuringSchdulingIgnoredDuringExecution 字段实现的。

**添加标签**

```s
kubectl label node node1.k8s availability-zone=zone1
kubectl label node node1.k8s share-type=dedicated
kubectl label node node2.k8s availability-zone=zone2
kubectl label node node2.k8s share-type=shared
```

**指定优先级节点亲和性**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pref
spec: 
  template:
    ...
    spec:
      affinity:
        nodeAffinity:
          prefereddDuringSchdulingIgnoredDuringExecution:
          - weight: 80
            preference:
            - matchExpressions:
              - key: availability-zone
                operator: In
                values:
                - zone1
          - weight: 20
            preference:
            - matchExpressions:
              - key: share-type
                operator: In
                values:
                - dedicated
```
上面描述一个节点亲和优先级，并不是强制要求。想要 Pod 调度到含有 availability-zone=zone1
和 share-type=dedicated 的节点上。第一个优先级相对重要，weight=80，第二个相对不重要，weight=20。

**了解优先级是如何工作的**

如果集群中包含很多节点，上面的 deployment，将会把节点分成四种。2 个标签都包含的，优秀级最高；
只包含 weight=80 的标签节点次之；然后只是包含 weight=20 的节点；剩下的节点排在最后。

**在一个包含 2 个节点的集群中部署**

如果在只有 node1 和 node2 的集群中部署一个 replica=5 的 Deployment，Pod 更多部署到 node1，
有极少数调度到 node2 上。这是因为 kube-schduler 除了节点亲和的优先级函数，
还有其他函数来决定 Pod 调度到哪里。其中之一就是 Selector SpreadPriority 函数，
此函数保证同一个 ReplicaSet 或者 Service 的 Pod，将分散到不同节点上。
避免单个节点失效而出现整个服务挂掉。

# 3. 使用亲和/反亲和部署 Pod

刚才描述了 Pod 和节点之间的亲和关系，也有 Pod 与 Pod 之间的亲和关系，
例如前端 Pod 和后端 Pod，尽可能部署较近，减少网络时延。

## 3.1 使用 Pod 亲和将多个 Pod 部署到同一个节点上

部署 1 个后端 Pod 和 5 个前端 Pod。先部署后端：
```s
kubectl run backend -l app=backend --image busybox --sleep 9999
```
这里没什么区别，只是给 Pod 加了个标签，app=backend。

**在 Pod 中指定亲和性**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec: 
  replicas: 5
  template:
    ...
    spec:
      affinity:
        podAffinity:
          requireddDuringSchdulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector: 
              matchLabels:
                app: backend
```
这里强制要求，frontend 的 Pod 被调度到和其他包含 app=backend 标签的 Pod 所在的相同节点上。
这是通过 topologyKey 指定的。

**了解调度器如何使用 pod 亲和规则**

上面的应用部署后，5 个前端 Pod 和 1 个后端 Pod 都在 node2 上。
假设后端 Pod 被误删，重新调度后，依旧在 node2 上。
虽然后端 Pod 本身没有定义任何亲和性规则，如果调度到其他节点，即会打破已有的亲和规则。

## 3.2 将 Pod 部署到同一个机柜、可用性区域或者地理地域

**同一个可用性区域协调部署**

如果想要 Pod 在同一个 available zone 部署应用，
可以将 topologyKey 设置成 failure-domain.beta.kubernetes.io/zone。

**同一个地域协调部署**
如果想要 Pod 在同一个 available zone 部署应用，
可以将 topologyKey 设置成 failure-domain.beta.kubernetes.io/region。

**了解 topologyKey 如何工作**
topologyKey 的工作方式很简单，上面提到的三个属性并没有什么特别。可以设置成任意你想要的值。

当调度器决定决定 Pod 调度到哪里时，首先检查 podAffinity 配置，选择满足条件的 Pod，
接着查询这些 Pod 运行在那些节点上。特别寻找能匹配 podAffinity 配置的 topologyKey 的节点。
接着，会优先选择所有标签匹配的 Pod 的值的节点。

## 3.3 Pod 优先级亲和性
与节点亲和一样，pod 之间也可以用 prefereddDuringSchdulingIgnoredDuringExecution。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    ...
    spec:
      affinity:
        podAffinity:
          prefereddDuringSchdulingIgnoredDuringExecution:
          - weight: 80
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector: 
                matchLabels:
                  app: backend
```
和 nodeAffinity 一样，设置了权重。也需要设置 topologyKey 和 labelSelector。

## 3.4 Pod 反亲和分开调度

同理，有 nodeAntiAffinity，也有 podAntiAffinity。
这将导致调度器永远不会选择包含 podAntiAffinity 的 Pod 所在的节点。

**使用反亲和分散 Pod**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAffinity:
         requiredDuringSchdulingIgnoredDuringExecution:
            topologyKey: kubernetes.io/hostname
            labelSelector: 
              matchLabels:
                app: frontend
```
上面的 Deployment 创建后，5 个实例应当分布在 5 个不同节点上。

**理解 Pod 反亲和优先级**

在这种情况下，可能使用软反亲和策略（prefereddDuringSchdulingIgnoredDuringExecution）。
毕竟 2 个前端 Pod 在同一个节点上也不是什么问题。如果运行在一个节点上出现问题，
那么使用 requiredDuringSchdulingIgnoredDuringExecution 就比较合适了。
与 Pod 亲和一样，topologyKey 决定了 Pod 不能被调度的范围。