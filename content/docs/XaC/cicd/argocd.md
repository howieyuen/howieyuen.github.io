---
author: howieyuen
date: 2021-10-21
title: 使用 ArgoCD ApplicationSet 管理多集群
tags: [ArgoCD]
categories: [CI/CD]
---

> 本文翻译自
> [Manage More Clusters with Less Hassle, with ArgoCD Application Sets - Jonathan and Kshama](https://kccncna2021.sched.com/event/lUzj/manage-more-clusters-with-less-hassle-with-argo-cd-application-sets-jonathan-west-red-hat-kshama-jain-independent-contributor?iframe=no&w=&sidebar=yes&bg=no)
> 
 
## ApplicationSet

Applicationset 是一种新的Kubernetes自定义资源（CR）和控制器，它与现有的 ArgoCD 一起工作。
Applicationset 是 ArgoCD 应用程序的工厂：`ApplicationSet` CR 描述了创建/管理的应用程序，ArgoCD 负责部署它们。

特征：
- 将大量 ArgoCD 应用程序作为单个单元管理
- 你集群的 Deployment 可以来自各种数据源的自动化和自定义：
  - 例如，随着新的集群添加到基础架构中，添加了新的 Git Repo，将添加新的 Repo 到 Github/Gitlab 等
- 对外变更自动作出反应
  - 基于外部事件的一致性是快速且可自定义的：在飞行中创建/更新/删除应用程序
- Applicationset 基于 ArgoCD 应用，可利用 ArgoCD 带来的所有能力

## GitOps 介绍

- Git 来源于事实
- 所需的软件状态在 Git 仓库中以文件形式存储
- GitOps 代理将确保所需的软件状态已在运行时环境中部署

## ArgoCD 介绍

- 基于 GitOps 的 K8S 的持续性传输工具。
- 使用 Git Repo 作为事实来源。
- 声明式持续性交付工具。
- 支持各种配置管理工具（ksonnet、jsonnet、kustomize 和 helm）。
- 作为 K8S 控制器和 CRD 实现。
- 提供具有强大的资源管理功能的 Kubernetes 仪表板。
- 自动同步应用程序的当前实时状态到期望状态。

## ArgoCD 关键特性

- 强大、实时的网络 UI
- 支持多个配置管理/模板工具
- 多集群支持
- SSO 集成
- 资源健康评估
- 自动配置漂移检测和差异
- 自动或手动将应用程序同步到其期望状态
- 自动化 CLI 和 CI 集成
- 审计活动
- Prometheus 度量标准
- GnuPG 签名验证
- 同步的前置/后置钩子
- 多租户和 RBAC 授权策略

## ArgoCD Application 自定义资源

定义了 ArgoCD 做什么的核心实例是 Application 的 CR。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
	name: guestbook
	namespace: argocd
spec:
	project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

ArgoCD 应用程序就像 Kubernetes 集群和 Git 之间的“胶水”：“在 Git 中定义这些 K8S 资源，并将它们应用于我的集群，并保持同步。”
但请注意，只有一个源（Git 存储库），而且只有一个目标（某个集群的某个命名空间）。

## ApplicationSet 介绍

与 ArgoCD Application 不同，只能将单个 Git 存储库路径连接到单个 Kubernetes 集群命名空间，Applicationset 允许你定义许多这样的连接（作为 ArgoCD Application），并将全部管理为单个单元。

ApplicationSet Controller 是开源的，是 ArgoCD 子项目，托管在 Argoproj-Labs 组织中，该项目由许多不同人员的贡献构建，包括来自Red Hat、Snyk、Intuit、MLB、阿里云等个人贡献。

ApplicationSet Controller 需要现有的 ArgoCD 安装，并与其一起工作。

## ApplicationSet 自定义资源 (CR)

generator 字段引用指定一个生成器，它负责生成模板参数。
模板参数是键值对，它将被替换为模板：
- cluster：engineering-dev
- URL：https//kubernetes.default.svc
参数呈现为模板的相应 `{{parameter name}}` 字段。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
name: guestbook
spec:
  generators:
  - list:
    elements:
    - cluster: engineering-dev
  		url: https://kubernetes.default.svc
  template:
  	metadata:
    	name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        path: examples/list-generator/guestbook/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: guestbook
```

## List Generator

- **是什么**：键值的简单列表，可直接替换为模板，生成 Application。
- **为什么**：最简单的生成器。 擅长初始化实验应用程序和非常基本的部署方案。

虽然比同等的 ArgoCD Application 的 YAML 要少，但仍然不如其他生成器强大。
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
name: guestbook
spec:
  generators:
  - list:
    elements:
    - cluster: engineering-dev
  		url: https://kubernetes.default.svc
    - cluster: engineering-prod
     	url: https://kubernetes.default.svc
  template:
  	metadata:
    	name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        path: examples/list-generator/guestbook/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: guestbook
```

## Generator

生成器负责生成参数，然后将其呈现为模板：ApplicationSet 的字段。
生成器主要基于它们用于生成模板参数的数据源：
- 列表生成器从文字列表中提供了一组参数
- 集群生成器使用 ArgoCD 集群列表作为数据源
- Git 生成器使用 Git 存储库中的文件/目录
- ...

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: guestbook
spec:
  generators:
    - list:
      elements:
        - cluster: engineering-dev
        	url: https://kubernetes.default.svc
        - cluster: engineering-prod
        	url: https://kubernetes.default.svc
  template:
  # (...)
```

## Template

ApplicationSet 规约的模板字段用于生成 ArgoCD Application 资源。

通过将来自生成器的参数与模板的字段（通过 `{{values}}`）组合来创建 ArgoCD Application，并且从该具体应用程序资源产生并应用于集群。

模板子字段直接对应于 [ArgoCD Application 资源](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#applications)的规约（[示例](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/application.yaml)）。

```yaml
template:
  metadata:
  	name: '{{cluster}}-guestbook'
  spec:
    source:
      repoURL: https://github.com/infra-team/cluster-deployments.git
      targetRevision: HEAD
      path: guestbook/{{cluster}}
    destination:
      server: '{{url}}'
      namespace: guestbook
```

## 合并

对父 ApplicationSet 的更改自动应用所有子 ArgoCD Application。
- 这允许你从单个 ApplicationSet 管理许多 Application 的内容。

ApplicationSet Controller 负责重新调整 ApplicationSet。基于应用程序集的模板字段的内容，控制器生成一个或多个 Application。
- 这是应用程序控制器的责任结束的位置。

ArgoCD 负责其部署 Application 资源。
- 从 Git 读取 kubernetes 资源
- 将这些资源应用于目标集群/命名空间

## Cluster 生成器

- **是什么**：自动读取使用 ArgoCD 的定义的集群列表，并给每个集群创建 Application。
- **为什么**：应用添加到 ArgoCD 时自动部署 ArgoCD Application。

你同样可以使用 Cluster Annotation 来[定位集群的子集](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/)。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: guestbook
spec:
  generators:
    - clusters: {}
    template:
      metadata:
      	name: '{{name}}-guestbook'
      spec:
        project: "default"
        source:
          repoURL: https://github.com/argoproj/argocd-example-apps/
          targetRevision: HEAD
          path: guestbook
        destination:
          server: '{{server}}'
          namespace: guestbook
```

## Git Directory Generator

- **是什么**：扫描 Git 仓库的内容，匹配[目录](https://github.com/argoproj-labs/applicationset/tree/master/examples/git-generator-directory)，并创建 ArgoCD Application。
- **为什么**：对于部分小伙伴，偏向于用一个 Git 仓库来定义多个 workload、component 和 microservice 等 k8s 资源，此生成器将会自动定位 Git 仓库。Git 上任何增删操作都会自动检测并映射到 k8s 中。
对于相反的方式，每个应用程序组件/微服务都有自己的存储库，请参阅 SCM Provider 生成器。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: cluster-addons
spec:
  generators:
  - git:
      repoURL: https://github.com/argoproj-labs/applicationset.git
      revision: HEAD
      directories:
      - path: examples/git-generator-directory/cluster-addons/*
  template:
     metadata:
      name: '{{path.basename}}'
      spec:
        project: default
        source:
          repoURL: https://github.com/argoproj-labs/applicationset.git
          targetRevision: HEAD
          path: '{{path}}'
        destination:
          server: https://kubernetes.default.svc
          namespace: '{{path.basename}}'
```

## SCM Provider Generator

- **是什么**：扫描 GitHub/GitLab 上某个组织下的仓库列表，给满足匹配的仓库创建 Application。
- **为什么**：部分小伙伴偏向于拆分应用和微服务的 Deployment 到同一个 GitHub/GitLab 的组织下不同仓库。此生成器将会自动扫描组织，并将该组织下的所有应用部署到目标位置。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: guestbook
spec:
  generators:
  - scmProvider:
    github:
      organization: argoproj
      cloneProtocol: https
    filters:
    - repositoryMatch: example-apps
  template:
    metadata:
    	name: '{{ repository }}-guestbook'
    spec:
      project: "default"
      source:
        repoURL: '{{ url }}'
        targetRevision: '{{ branch }}'
      	path: guestbook
      destination:
        server: https://kubernetes.default.svc
        namespace: guestbook
```

## Git File Generator

- **是什么**：扫描 Git 仓库中的内容，并给满足 [JSON/YAML](https://github.com/argoproj-labs/applicationset/tree/master/examples/git-generator-files-discovery) 匹配的文件创建 ArgoCD Application。
- **为什么**：定义一系列配置文件，来描述具体要创建的 Application。通过 Git Commit 提供对单个字段的细粒度控制。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: guestbook
spec:
  generators:
  - git:
      repoURL: https://github.com/argoproj-labs/applicationset.git
      revision: HEAD
      files:
        - path: |
            "examples/git-generator-files-discovery/cluster-config/**/config.json"
  template:
    metadata:
    	name: '{{cluster.name}}-guestbook'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        path: "examples/git-generator-files-discovery/apps/guestbook"
      destination:
        server: https://kubernetes.default.svc
        namespace: guestbook
```

## Matrix Generator

- **是什么**：将两个生成器合并，因此也是合并了二者的优点
- **为什么**：高灵活性地合并生成器，例如：
- 将 Git 中每个应用部署到所有 ArgoCD 集群
  - Git Repository：Git 仓库中的所有应用
  - Cluster：每个 ArgoCD 集群
- 扫描 GitHub 组织并部署仓库到 Git 中的 YAML 文件定义的环境列表。
  - Git File：对于 repo 中每个匹配的 YAML 文件，将内容转换为参数
  - SCM Provider：扫描整个组织并转换为参数

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: cluster-git
spec:
  generators:
  - matrix:
    generators:
      - git:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        revision: HEAD
        directories:
        - path: examples/matrix/cluster-addons/*
      - clusters:
          selector:
            matchLabels:
            	argocd.argoproj.io/secret-type: cluster
  template:
    metadata:
    	name: '{{path.basename}}-{{name}}'
    spec:
      project: '{{metadata.labels.environment}}'
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
```

## 集群决策资源生成器

- **是什么**："外部资源"的逻辑，是用集群外的 CR 存放应用
- **为什么**：也许是最复杂的生成器（但也是最强大的）。使用它需要编写你自己的自定义控制器，定义 CRD，并使用你的控制器修改 CR 的状态字段以表明应将 Application 部署到哪些集群。
支持与 Open Cluster Management 项目的 [Placement Rule CR](https://open-cluster-management.io/concepts/placement/#select-clusters-in-managedclusterset) 集成（这些人贡献了生成器）。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: book-import
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: ocm-placement
        name: test-placement
        requeueAfterSeconds: 30
  template:
    metadata:
    	name: '{{clusterName}}-book-import'
    spec:
      project: "default"
      source:
        repoURL: |
        https://github.com/open-cluster-management/application-samples.git
        targetRevision: HEAD
        path: book-import
      destination:
        name: '{{clusterName}}'
        namespace: bookimport
```

## ApplicationSet 在 Red Hat

ApplicationSet 与 ArgoCD 一起编译到 Red Hat OpenShift GitOps。了解更多有关如何在 OpenShift 上使用 ArgoCD 部署 ApplicationSet：
- [OpenShift GitOps 的 ApplicationSet 介绍 —— Jonathan West & Dewan Ahmed](https://cloud.redhat.com/blog/an-introduction-to-applicationsets-in-openshift-gitops) 
- [ApplicationSet 入门 —— Christian Hernandez](https://cloud.redhat.com/blog/getting-started-with-applicationsets)

## ApplicationSet 在 Intuit

- 自动化跨多个 kubernetes 集群定制的插件安装。
- 用于 Argo 部署安装的 Git File Generator。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: argo-rollouts
spec:
  generators:
    - git:
      repoURL: 'https://github.com/argoproj-labs/applicationset.git'
      revision: HEAD
      files:
      	- path: argo-rollouts/env/*/config.json
  template:
    metadata:
    	labels:
    		appset: argo-rollouts
    		region: '{{region}}'
    	name: 'argo-rollouts.{{name}}'
    spec:
      destination:
        namespace: argo-rollouts
        server: '{{server}}'
        info:
          - name: cluster-url
          	value: 'kubernetes.default.svc'
      project: argo-rollouts
      source:
        path: 'argo-rollouts/env/{{name}}'
        repoURL: 'https://github.com/argoproj-labs/applicationset.git'
        targetRevision: HEAD
```

## 快速链接

- 入门
  - [ApplicationSet 入门](https://argocd-applicationset.readthedocs.io/en/stable/Geting-Started/)：介绍和快速入门
- 了解更多
  - [ApplicationSet 文档](https://argocd-applicationset.readthedocs.io/en/stable/)：模板、生成器、架构等等
- 加入我们
  - GitHub：[ArgoCD ApplicationSet](https://github.com/argoproj-labs/applicationset) 项目
  - CNCF Slack：[#argo-cd-appset](https://argoproj.github.io/community/join-slack/)
