---
author: Yuan Hao
date: 2020-12-29
title: 鉴权机制
tags: [kube-apiserver, authorization]
categories: [Kubernetes]
weight: 2
---

在客户端请求通过认证后，会进入鉴权阶段，kube-apiserver 同样支持多种鉴权机制，并支持同时开启多个鉴权模块。
如果开启多个鉴权模块，则按照顺序执行鉴权模块，排在前面的鉴权模块有较高的优先级来允许或者拒绝请求。
只要有一个鉴权模块通过，则鉴权成功。

<!--more-->

# 鉴权概述

kube-apiserver 目前提供了 6 种鉴权机制：
- AlwaysAllow：总是允许
- AlwaysDeny：总是拒绝
- ABAC：基于属性的访问控制（Attribute-Based Access Control）
- Node：节点鉴权，专门鉴权给 kubelet 发出的 API 请求
- RBAC：基于角色额的访问控制（Role-Based Access Control）
- Webhook：基于 webhook 的一种 HTTP 回调机制，可以进行远程鉴权管理

在学习鉴权之前，有三个概念需要补齐：
- Decision：决策状态
- Authorizer：鉴权接口
- RuleResolver：规则解析器

## Decision：决策状态

Decision 决策状态类似于认证中的 true 和 false，用于决定是否鉴权成功。
鉴权支持三种 Decision 决策状态，例如鉴权成功，则返回 DecisionAllow，代码如下：

```go
// staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go
type Decision int

const (
	DecisionDeny Decision = iota
	DecisionAllow
	DecisionNoOpinion
)
```
- DecisionDeny：拒绝该操作
- DecisionAllow：允许该操作
- DecisionNoOpinion：表示无明显意见允许或拒绝，继续执行下一个鉴权模块

## Authorizer：鉴权接口

每个鉴权模块都要实现该接口的方法，代码如下：

```go
// staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go
type Authorizer interface {
	Authorize(ctx context.Context, a Attributes) (authorized Decision, reason string, err error)
}
```

Authorizer 接口定义了 Authorize 方法，该方法接收一个 Attribute 参数。
Attributes 是决定鉴权模块从 HTTP 请求中获取鉴权信息方法的参数，它是一个方法集合的接口，
例如 GetUser、GetVerb、GetNamespace、GetResource 等鉴权信息方法。
如果鉴权成功，Decision 状态变成 DecisionAllow，
如果鉴权失败，Decision 状态变成 DecisionDeny，并返回失败的原因。

Attributes 定义如下：

```go
// staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go
type Attributes interface {
	GetUser() user.Info

	GetVerb() string

	IsReadOnly() bool

	GetNamespace() string

	GetResource() string

	GetSubresource() string

	GetName() string

	GetAPIGroup() string

	GetAPIVersion() string

	IsResourceRequest() bool

	GetPath() string
}
```

## RuleResolver：规则解析器

鉴权模块通过 RuleResolver 解析规则，定义如下：

```go
// staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go
type RuleResolver interface {
	RulesFor(user user.Info, namespace string) ([]ResourceRuleInfo, []NonResourceRuleInfo, bool, error)
}
```

RuleResolver 接口定义的 RulesFor 方法，所有鉴权模块都要实现。
RulesFor 方法接受 user 用户信息和 namespace 命名空间参数，解析出规则列表并返回。
规则列表分成两种：
- ResourceRuleInfo：资源类型的规则列表，例如 /api/v1/pods 的资源接口
- NonResourceRuleInfo：非资源类型的规则列表，例如 /healthz 的非资源接口

以 ResourceRuleInfo 资源类型为例，定义如下：

```go
// staging/src/k8s.io/apiserver/pkg/authorization/authorizer/interfaces.go
type DefaultResourceRuleInfo struct {
	Verbs         []string
	APIGroups     []string
	Resources     []string
	ResourceNames []string
}
```

Pod 资源规则列表示例如下，其中通配符（*）表示匹配所有，该规则表示用户对所有资源版本的 Pod 拥有所有操作权限，
即：get、list、watch、create、update、patch、delete、deletecollection。

```go
resourcesRules: []authorizer.ResourceRuleInfo {
    &authorizer.DefaultResourceRuleInfo {
        Verbs:     []string{"*"}
        APIGroups: []string{"*"}
        Resources: []string{"pods"}
    }
}
```

每一种鉴权机制实例化后，成为一个鉴权模块，被封装在 http.Handler 函数中，他们接受组件或者客户端的请求并鉴权。
假设 kube-apiserver 启用了 Node 鉴权模块 和 RBAC 鉴权模块。
请求会进入 Authorization Handler 函数，该函数会遍历已经启用的鉴权模块列表，按照顺序执行每个鉴权模块，
例如在 Node 鉴权模块返回 DecisionNoOpinion 时，会继续执行 RBAC 鉴权模块。代码如下：

```go
// staging/src/k8s.io/apiserver/pkg/authorization/union/union.go
func (authzHandler unionAuthzHandler) Authorize(ctx context.Context, a authorizer.Attributes) (authorizer.Decision, string, error) {
	var (
		errlist    []error
		reasonlist []string
	)

	for _, currAuthzHandler := range authzHandler {
		decision, reason, err := currAuthzHandler.Authorize(ctx, a)

		if err != nil {
			errlist = append(errlist, err)
		}
		if len(reason) != 0 {
			reasonlist = append(reasonlist, reason)
		}
		switch decision {
		case authorizer.DecisionAllow, authorizer.DecisionDeny:
			return decision, reason, err
		case authorizer.DecisionNoOpinion:
			// continue to the next authorizer
		}
	}

	return authorizer.DecisionNoOpinion, strings.Join(reasonlist, "\n"), utilerrors.NewAggregate(errlist)
}
```

# ABAC 鉴权

## 概述

ABAC 授权器是基于属性的访问控制（Attributed-Based Access Control，ABAC）定义了访问控制范例，
其中通过属性组合在一起的策略来向用户授予操作权限。

## 启用

- `--authorization-mode=ABAC`：启用 ABAC 授权器
- `--authorization-policy-file`：基于 ABAC 模式，指定策略文件，
  该文件使用 [JSON Lines](https://jsonlines.org/) 格式描述，每行都是一个策略对象。

ABAC 模式策略文件定义：

```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "Deck", "namespace": "*", "resource": "*", "apiGroup": "*"}}
```

上面的策略，Deck 用户可以对所有资源做任何操作。

## 实现

```go
// pkg/auth/authorizer/abac/abac.go
func (pl PolicyList) Authorize(ctx context.Context, a authorizer.Attributes) (authorizer.Decision, string, error) {
	for _, p := range pl {
		if matches(*p, a) {
			return authorizer.DecisionAllow, "", nil
		}
	}
	return authorizer.DecisionNoOpinion, "No policy matched.", nil
}
```

进行 ABAC 策略授权时，遍历所有策略，通过 `matches` 函数判断是否匹配，有一个策略满足，返回 `DecisionAllow` 决策状态。

此外，ABAC 的规则解析器会根据每个策略，将资源类型的规则列表（ResoureceRuleInfo）和非资源类型的规则列表（NonResourceInfo）
都设置为该用户有权限操作的资源版本、资源和资源操作方法。
代码如下：

```go
// pkg/auth/authorizer/abac/abac.go
func (pl PolicyList) RulesFor(user user.Info, namespace string) ([]authorizer.ResourceRuleInfo, []authorizer.NonResourceRuleInfo, bool, error) {
	var (
		resourceRules    []authorizer.ResourceRuleInfo
		nonResourceRules []authorizer.NonResourceRuleInfo
	)

	for _, p := range pl {
		if subjectMatches(*p, user) {
			if p.Spec.Namespace == "*" || p.Spec.Namespace == namespace {
				if len(p.Spec.Resource) > 0 {
					r := authorizer.DefaultResourceRuleInfo{
						Verbs:     getVerbs(p.Spec.Readonly),
						APIGroups: []string{p.Spec.APIGroup},
						Resources: []string{p.Spec.Resource},
					}
					var resourceRule authorizer.ResourceRuleInfo = &r
					resourceRules = append(resourceRules, resourceRule)
				}
				if len(p.Spec.NonResourcePath) > 0 {
					r := authorizer.DefaultNonResourceRuleInfo{
						Verbs:           getVerbs(p.Spec.Readonly),
						NonResourceURLs: []string{p.Spec.NonResourcePath},
					}
					var nonResourceRule authorizer.NonResourceRuleInfo = &r
					nonResourceRules = append(nonResourceRules, nonResourceRule)
				}
			}
		}
	}
	return resourceRules, nonResourceRules, false, nil
}
```

# RBAC 鉴权

## 概述

RBAC 是基于角色的访问控制（Role-Based Access Control），是根据组织中用户的角色来控制对计算机或者网络资源访问的方法。
也是目前使用最为广泛的鉴权模型。在 RBAC 鉴权模块中，权限与角色相关联，形成了用户——角色——权限的鉴权模型。
用户通过加入某些角色从而的得到这些角色的操作权限，极大地简化了权限管理。RBAC 的鉴权图如下所示：

{{< mermaid class="text-center">}}
graph LR;
    A[用户] --> B[角色]
    B --> C[权限]
{{< /mermaid >}}

**API 对象**

RBAC API 声明了 4 种 Kubernetes 对象：Role、ClusterRole、RoleBinding、ClusterRoleBinding。

**Role 和 ClusterRole**

Role 和 ClusterRole 中包含一组代表相关权限的规则，这些权限是单纯的累加关系，不存在角色某种操作的规则。
这 2 种资源类型不同，是因为 kubernetes 的对象要么是命名空间作用域（Namespace Scope），要么是集群作用域（Cluster Scope）。

数据结构如下：
{{< mermaid class= "text-center">}}
classDiagram
class Role{
    +metav1.TypeMeta
    +metav1.ObjectMeta
    +Rules []PolicyRule
}

class ClusterRole{
    +metav1.TypeMeta
    +metav1.ObjectMeta
    +Rules []PolicyRule
}

class PolicyRule{
    +Verbs []string
    +APIGroups []string
    +Resources []string
    +ResourceNames []string
}
Role "1"-->"n" PolicyRule
ClusterRole "1"-->"n" PolicyRule
{{< /mermaid >}}

- Role：总是用来设置某个命名空间里的发个文权限。
- ClusterRole：是集群作用域的资源。
- Rule：规则相当于操作权限，控制资源的操作方法（即 Verbs）

**RoleBinding 和 ClusterRoleBinding**

数据结构如下：
{{< mermaid class="text-center" >}}
classDiagram

class RoleBinding{
    +metav1.TypeMeta
    +metav1.ObjectMeta
    +Subjects []Subeject
    +RoleRef RoleRef
}

class ClusterRoleBinding{
    +metav1.TypeMeta
    +metav1.ObjectMeta
    +Subjects []Subeject
    +RoleRef RoleRef
}

class Subject{
    +Kind string
    +APIGroup string
    +Name string
    +Namespace string
}

class RoleRef{
    +APIGroup string
    +Kind string
    +Name string
}

RoleBinding "1" -->"n" Subject
RoleBinding "1"-->"1" RoleRef
ClusterRoleBinding "1"-->"n" Subject
ClusterRoleBinding "1"-->"1" RoleRef
{{< /mermaid >}}

- RoleRef：被授予权限的角色的引用
- Subject：主体可以是用户、组或者服务账户
- RoleBinding：将角色中定义的权限授予一个或者一组用户，只能用于某一个命名空间的权限
- clusterRoleBinding：将集群角色的中定义的权限首页一个或者一组用户没，可用于集群范围的权限

## 启用

kube-apiserver 通过指定以下参数启用 Node 鉴权
- `--authorization-mode=RBAC`

## 实现

kube-apiserver 通过表示用户、操作、角色、角色绑定来描述 RBAC 的关系。模型图如下：

```
+--------+        +-------------+         +--------+         +-------------+
|  User  |        | RoleBinding |         |  Role  |         |  Operation  |
+--------+        +-------------+         +--------+         +-------------+
| User-A |<-------|             |-------->|        |-------->| Operation-A |
+--------+        |  Binding-A  |         | Role-A |         +-------------+
| User-B |<-------|             |-------->|        |-------->| Operation-B |
+--------+        +-------------+         +--------+         +-------------+
| User-C |<-------|             |-------->|        |-------->| Operation-C |
+--------+        |  Binding-B  |         | Role-B |         +-------------+
| User-D |<-------|             |-------->|        |-------->| Operation-D |
+--------+        +-------------+         +--------+         +-------------+
```

Role-A 角色拥有 Operation-A 和 Operation-B 的权限，Binding-A 将用户 User-A 与角色 Role-A 绑定，
因此 User-A 就有了 Operation-A 和 Operation-B 的权限，但没有 Operation-C 和 Operation-D 的权限。

```go
// plugin/pkg/auth/authorizer/rbac/rbac.go
func (r *RBACAuthorizer) Authorize(ctx context.Context, requestAttributes authorizer.Attributes) (authorizer.Decision, string, error) {
    ruleCheckingVisitor := &authorizingVisitor{requestAttributes: requestAttributes}
    // ruleCheckingVisitor.visit -> RulesAllow -> 各种 match 函数验证
	r.authorizationRuleResolver.VisitRulesFor(requestAttributes.GetUser(), requestAttributes.GetNamespace(), ruleCheckingVisitor.visit)
	if ruleCheckingVisitor.allowed {
		return authorizer.DecisionAllow, ruleCheckingVisitor.reason, nil
    }
    ...
	return authorizer.DecisionNoOpinion, reason, nil
}
```

进行 RBAC 鉴权时，首先调用 `ruleCheckingVisitor.visit` 验证授权，该函数返回的 allowed 字段为 true，表示授权成功。
`ruleCheckingVisitor.visit` 会调用 RBAC 的 `RulesAllows` 函数，该函数是实际验证函数的合集。
鉴权的原理如下所示：

{{< mermaid class="text-center" >}}
graph LR
A{IsResourceRequest} --> |Yes| B[VerbMacthes]
B --> C[APIGroupMacthes]
C --> D[ResourceMacthes]
D --> E[ResourceNameMacthes]
A -->|No| F[VerbMatchs]
F --> G[NonResourceMacthes] 
{{< /mermaid >}}

## 内置 ClusterRole

介绍内置角色之前，先了解下 kube-apiserver 的内置权限，内置的角色会引用内置权限，代码示例如下：

```go
// plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go
var (
	Write      = []string{"create", "update", "patch", "delete", "deletecollection"}
	ReadWrite  = []string{"get", "list", "watch", "create", "update", "patch", "delete", "deletecollection"}
	Read       = []string{"get", "list", "watch"}
	ReadUpdate = []string{"get", "list", "watch", "update", "patch"}
)
```

- Write：只写
- ReadWrite：读写
- Read：只读
- ReadUpdate：只读和更新

kube-apiserver 在启动时，会默认创建内置角色，例如：cluster-admin，它拥有最高权限；定义如下：

```go
// plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go
{
    // a "root" role which can do absolutely anything
    ObjectMeta: metav1.ObjectMeta{Name: "cluster-admin"},
    Rules: []rbacv1.PolicyRule{
        rbacv1helpers.NewRule("*").Groups("*").Resources("*").RuleOrDie(),
        rbacv1helpers.NewRule("*").URLs("*").RuleOrDie(),
    },
},
```

cluster-admin 的定义中，将资源类型和非资源类型都设置为通配符（*），匹配所有版本和资源，与集群角色 `system:master` 绑定。
在 `plugin/pkg/auth/authorizer/rbac/bootstrappolicy` 目录下，有关所有内置的角色和权限配置信息，具体可以分为三类：
API 发现角色、面向用户角色和面向组件角色。

**Discover Roles**

无论是经过身份验证的还是未经过身份验证的用户，默认的角色绑定都授权他们读取被认为是可安全地公开访问的 API（包括 CustomResourceDefinitions）。
如果要禁用匿名的未经过身份验证的用户访问，请在 API 服务器配置中中添加 --anonymous-auth=false 的配置选项。

ClusterRole       | ClusterRoleBinding   | 说明
----------------- | -------------------- | ----
system:basic-user | system:authenticated | 允许用户以只读的方式去访问他们自己的基本信息
system:discovery  | system:authenticated | 允许以只读方式访问 API discovery endpoints
system:public-info-viewer | system:authenticated 和 system:unauthenticated | 允许对集群的非敏感信息进行只读访问

**User-facing Roles**

面向用户的角色，包含超级用户角色（cluster-admin），即在集群范围内授权的角色，
以及那些使用 使用 RoleBinding（admin、edit、view）在特定命名空间空间中授予的角色。

ClusterRole   | ClusterRoleBinding | 说明
------------- | ------------------ | ----
cluster-admin | system:masters     | 超级用户权限，允许对任何资源执行任何操作
admin         | None               | 允许管理员访问权限，旨在使用 RoleBinding 在命名空间内执行授权。如果在 RoleBinding 中使用，则可授予对命名空间中的大多数资源的读/写权限，包括创建角色和角色绑定的能力。但是它不允许对资源配额或者命名空间本身进行写操作。
edit          | None               | 允许对命名空间的大多数对象进行读/写操作。它不允许查看或者修改角色或者角色绑定。不过，此角色可以访问 Secret，以命名空间中任何 ServiceAccount 的身份运行 Pods，所以可以用来了解命名空间内所有服务账号的 API 访问级别。
view          | None               | 允许对某个命名空间内大部分的对象进行只读访问，但不允许查看 Role 或者 RoleBinding。由于可扩展行等原因，不允许查看 Secret 资源

**Component Roles**

- 核心组件角色：

ClusterRole   | ClusterRoleBinding | 说明
------------- | ------------------ | ----
system:kube-scheduler | system:kube-scheduler | 允许访问 scheduler 组件所需要的资源
system:volume-scheduler | system:kube-scheduler | 允许访问 kube-scheduler 组件所需要的卷资源
system:kube-controller-manager | system:kube-controller-manager | 允许访问控制器管理器组件所需要的资源。
system:node | None | 允许访问 kubelet 所需要的资源，包括对所有 Secret 的读操作和对所有 Pod 状态对象的写操作
system:node-proxier | system:kube-proxy | 允许访问 kube-proxy 组件所需要的资源

- 其他组件角色：

ClusterRole   | ClusterRoleBinding | 说明
------------- | ------------------ | ----
system:auth-delegator | 无 | 委托身份认证和鉴权检查。这种角色通常用在插件式 API 服务器上，以实现统一的身份认证和鉴权。
system:heapster       | 无 | 为 Heapster 组件（已弃用）定义的角色
system:kube-aggregator| 无 | 为 kube-aggregator 组件定义的角色
system:kube-dns | kube-dns | 为 kube-dns 组件定义的角色
system:kubelet-api-admin | 无 | 允许 kubelet API 的完全访问权限
system:node-bootstrapper | 无 | 允许访问执行 kubelet TLS 启动引导 所需要的资源
system:node-problem-detector | 无 | 为 node-problem-detector 组件定义的角色
system:persistent-volume-provisioner | 无 | 允许访问大部分动态卷驱动（dynamic volume driver）所需要的资源

**Controller Roles**

Kubernetes 控制器管理器 运行内建于 Kubernetes 控制面的控制器。
当使用 `--use-service-account-credentials` 参数启动时，
kube-controller-manager 使用单独的服务账号来启动每个控制器。 
每个内置控制器都有相应的、前缀为 `system:controller:` 的角色。
如果控制管理器启动时未设置 `--use-service-account-credentials`，
它使用自己的身份凭据来运行所有的控制器，该身份必须被授予所有相关的角色。
这些角色包括：

* `system:controller:attachdetach-controller`
* `system:controller:certificate-controller`
* `system:controller:clusterrole-aggregation-controller`
* `system:controller:cronjob-controller`
* `system:controller:daemon-set-controller`
* `system:controller:deployment-controller`
* `system:controller:disruption-controller`
* `system:controller:endpoint-controller`
* `system:controller:expand-controller`
* `system:controller:generic-garbage-collector`
* `system:controller:horizontal-pod-autoscaler`
* `system:controller:job-controller`
* `system:controller:namespace-controller`
* `system:controller:node-controller`
* `system:controller:persistent-volume-binder`
* `system:controller:pod-garbage-collector`
* `system:controller:pv-protection-controller`
* `system:controller:pvc-protection-controller`
* `system:controller:replicaset-controller`
* `system:controller:replication-controller`
* `system:controller:resourcequota-controller`
* `system:controller:root-ca-cert-publisher`
* `system:controller:route-controller`
* `system:controller:service-account-controller`
* `system:controller:service-controller`
* `system:controller:statefulset-controller`
* `system:controller:ttl-controller`

# Node 鉴权

## 概述

Node 鉴权，也成为节点鉴权，是一种专门针对 kubelet 发出的请求进行鉴权。
Node 鉴权机制基于 RBAC 授权机制实现，对 kubelet 组件进行基于 `system:node` 内置角色的权限控制。
`system:node` 内置角色的权限定义在 NodeRules 函数中，具体可以移步：

```go
// plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go
// NodeRules returns node policy rules, it is slice of rbacv1.PolicyRule.
func NodeRules() []rbacv1.PolicyRule {
	nodePolicyRules := []rbacv1.PolicyRule{
		// Needed to check API access.  These creates are non-mutating
		rbacv1helpers.NewRule("create").Groups(authenticationGroup).Resources("tokenreviews").RuleOrDie(),
		rbacv1helpers.NewRule("create").Groups(authorizationGroup).Resources("subjectaccessreviews", "localsubjectaccessreviews").RuleOrDie(),
		 ...
        }
}
```
 
在上面的代码中，允许 kubelet 执行以下操作：
读取操作：
- services
- endpoints
- nodes
- pods
- secrets、configmaps、pvc 以及绑定到 kubelet 节点的与 pod 相关的持久卷
 
写入操作：
- 节点和节点状态（启用 `NodeRestriction` 准入插件以限制 kubelet 只能修改自己的节点）
- pod 和 pod 状态（启用 `NodeRestriction` 准入插件以限制 kubelet 只能修改绑定到自身的 Pod）
- 事件
 
鉴权相关：
- 对于基于 TLS 的启动引导过程时使用的 certificationsigningrequests API 的读写权限
- 为委派的身份验证/鉴权检查创建 tokenreviews 和 subjectaccessreviews 的能力
 
## 启用

kube-apiserver 通过指定以下参数启用 Node 鉴权
- `--authorization-mode = Node,RBAC`

## 实现

```go
// plugin/pkg/auth/authorizer/node/node_authorizer.go
func (r *NodeAuthorizer) Authorize(ctx context.Context, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
	// 获取 node 角色信息
	nodeName, isNode := r.identifier.NodeIdentity(attrs.GetUser())
	if !isNode {
		// reject requests from non-nodes
		return authorizer.DecisionNoOpinion, "", nil
	}
	if len(nodeName) == 0 {
		// reject requests from unidentifiable nodes
		klog.V(2).Infof("NODE DENY: unknown node for user %q", attrs.GetUser().GetName())
		return authorizer.DecisionNoOpinion, fmt.Sprintf("unknown node for user %q", attrs.GetUser().GetName()), nil
	}

	// 特定资源的请求走这里
	if attrs.IsResourceRequest() {
		requestResource := schema.GroupResource{Group: attrs.GetAPIGroup(), Resource: attrs.GetResource()}
		switch requestResource {
		case secretResource:
			return r.authorizeReadNamespacedObject(nodeName, secretVertexType, attrs)
		case configMapResource:
			return r.authorizeReadNamespacedObject(nodeName, configMapVertexType, attrs)
		case pvcResource:
			if r.features.Enabled(features.ExpandPersistentVolumes) {
				if attrs.GetSubresource() == "status" {
					return r.authorizeStatusUpdate(nodeName, pvcVertexType, attrs)
				}
			}
			return r.authorizeGet(nodeName, pvcVertexType, attrs)
		case pvResource:
			return r.authorizeGet(nodeName, pvVertexType, attrs)
		case vaResource:
			return r.authorizeGet(nodeName, vaVertexType, attrs)
		case svcAcctResource:
			return r.authorizeCreateToken(nodeName, serviceAccountVertexType, attrs)
		case leaseResource:
			return r.authorizeLease(nodeName, attrs)
		case csiNodeResource:
			if r.features.Enabled(features.CSINodeInfo) {
				return r.authorizeCSINode(nodeName, attrs)
			}
			return authorizer.DecisionNoOpinion, fmt.Sprintf("disabled by feature gates %s", features.CSINodeInfo), nil
		}
	}
	// 非资源请求走这里
	if rbac.RulesAllow(attrs, r.nodeRules...) {
		return authorizer.DecisionAllow, "", nil
	}
	return authorizer.DecisionNoOpinion, "", nil
}
```

Node 鉴权时，通过 `r.identifier.NodeIdentity` 获取角色信息，
验证其是否为 `system:node` 内置角色，
nodeName 的格式为 `system:node<nodeName>` 。
资源类型请求，取出资源对象，内置了允许操作资源和操作方法。
非资源类型请求，走 `rbac.RulesAllow` 进行 RBAC 授权。

# Webhook

## 概述

webhook 是一种 HTTP 回调：当用户鉴权时，kube-apiserver 组件会查询外部的 webhook 服务。
该过程与 webhookTokenAuth 认证相似，但其中确认用户身份的机制不一样。
当客户端发送的认证请求到达 kube-apiserver 时，kube-apiserver 回调钩子方法，
将鉴权信息发给远程的 webhook 服务器进行认证，根据 webhook 服务返回的状态判断是否授权成功。

## 启用

- `--authorization-mode=Webhook`：启用 webhook 授权器
- `--authorization-webhook-config-file`：使用 kubeconfig 格式的 webhook 配置文件。

webhook 的配置文件如下：

```yaml
# Kubernetes API 版本
apiVersion: v1
# API 对象种类
kind: Config
# clusters 代表远程服务
clusters:
  - name: name-of-remote-authz-service
    cluster:
      # 对远程服务进行身份认证的 CA
      certificate-authority: /path/to/ca.pem
      # 远程服务的查询 URL。必须使用 'https'
      server: https://authz.example.com/authorize
# users 代表 API 服务器的 webhook 配置
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # webhook plugin 使用 cert
      client-key: /path/to/key.pem          # cert 所对应的 key
# kubeconfig 文件必须有 context，需要提供一个给 API 服务器
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authz-service
    user: name-of-api-server
  name: webhook
```

如上所示，users 指的是 kube-apiserver 本身，cluster 指的是远程 webhook 服务。

## 实现

```go
// staging/src/k8s.io/apiserver/plugin/pkg/authorizer/webhook/webhook.go
func (w *WebhookAuthorizer) Authorize(ctx context.Context, attr authorizer.Attributes) (decision authorizer.Decision, reason string, err error) {
	...
	// 尝试从缓存中查找该请求
	if entry, ok := w.responseCache.Get(string(key)); ok {
		r.Status = entry.(authorizationv1.SubjectAccessReviewStatus)
	} else {
		var (
			result *authorizationv1.SubjectAccessReview
			err    error
		)
		webhook.WithExponentialBackoff(ctx, w.retryBackoff, func() error {
			// 首次请求 webhook
			result, err = w.subjectAccessReview.Create(ctx, r, metav1.CreateOptions{})
			return err
		}, webhook.DefaultShouldRetry)
		...
		r.Status = result.Status
		// 写缓存
		if shouldCache(attr) {
			if r.Status.Allowed {
				w.responseCache.Add(string(key), r.Status, w.authorizedTTL)
			} else {
				w.responseCache.Add(string(key), r.Status, w.unauthorizedTTL)
			}
		}
	}
	switch {
	case r.Status.Denied && r.Status.Allowed:
		return authorizer.DecisionDeny, r.Status.Reason, fmt.Errorf("webhook subject access review returned both allow and deny response")
	case r.Status.Denied:
		return authorizer.DecisionDeny, r.Status.Reason, nil
	case r.Status.Allowed:
		return authorizer.DecisionAllow, r.Status.Reason, nil
	default:
		return authorizer.DecisionNoOpinion, r.Status.Reason, nil
	}
}
```

上面的代码可以看出，鉴权时，首先查询缓存，是否已经解析过此请求的鉴权。
如果有，直接使用该状态（Status），如果没有，create 一个 SubjectAccessView，
从远程的 webhook 服务器获取鉴权结果。`w.subjectAccessReview.Create` 是一个 POST 请求，
body 体中携带鉴权信息，r.Status.Allowed 为 true，表示鉴权成功。

此外，webhook 不支持规则列表解析，因为规则是由 webhook 服务器授权的。
所以，webhook 的规则解析器的资源类型列表（ResourceRulesInfo）和非资源类型规则列表（NonResourceRulesInfo）都设置为空。

```go
func (w *WebhookAuthorizer) RulesFor(user user.Info, namespace string) ([]authorizer.ResourceRuleInfo, []authorizer.NonResourceRuleInfo, bool, error) {
	var (
		resourceRules    []authorizer.ResourceRuleInfo
		nonResourceRules []authorizer.NonResourceRuleInfo
	)
	incomplete := true
	return resourceRules, nonResourceRules, incomplete, fmt.Errorf("webhook authorizer does not support user rule resolution")
}
```

# 参考资料

- [鉴权概述](https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/)
- [ABAC 鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/abac/)
- [RBAC 鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac)
- [Node 鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/node/)
- [webhook 模式](https://kubernetes.io/zh/docs/reference/access-authn-authz/webhook/)