---
title: DevOps vs GitOps
date: 2022-10-15
tags: [devops, gitops]
categories: [ops]
---

GitOps 和 DevOps 正迅速成为开发团队的黄金标准方法。DevOps 文化代表了从传统软件和技术开发的转变，传统的软件和技术开发涉及从构思和概念到发布的线性路径，并鼓励协作和快速反馈，而不是孤立地工作。在本文中，我们将仔细研究 GitOps 与 DevOps，以帮助确定 GitOps 是否适用于您的软件开发项目。

<!--more-->

# DevOps

从字面上来看，"DevOps" 一词是由英文 Development（开发）和 Operations（运维）组合而成，但它所代表的理念和实践要比这广阔的多。DevOps 涵盖了安全、协作方式、数据分析等许多方面。但它是什么呢？
DevOps 强调通过一系列手段来实现既快又稳的工作流程，使每个想法（比如一个新的软件功能，一个功能增强请求或者一个 bug 修复）在从开发到生产环境部署的整个流程中，都能不断地为用户带来价值。这种方式需要开发团队和运维团队密切交流、高效协作并且彼此体谅。此外，DevOps 还要能够方便扩展，灵活部署。有了 DevOps，需求最迫切的工作就能通过自助服务和自动化得到解决；通常在标准开发环境编写代码的开发人员也可与运维人员紧密合作，加速软件的构建、测试和发布，同时保障开发成果的稳定可靠。
一句话总结：DevOps 是文化、开发、运营和工具的结合体，可提高组织高速生产应用和服务的能力。

## 工作流

![](/posts/devops-vs-gitops/devops-workflow.png)

图中描述了 Dev 和 Ops 的角色任务是一个循环往复的过程：
- Dev 开始研发迭代（Plan），开始编写代码（Code），完成开始编译（Build）和测试（Test），最后发布（Release）
- Ops 拿到发布打包开始部署（Deploy），并运维（Operate）和监控（Monitor），并收集新的功能需求，特性增强以及 bug 修复等，形成下一个迭代安排（Plan）

# GitOps

GitOps 是一套使用 Git 来管理基础设施和应用配置的实践，而 Git 指的是一个开源版控制系统。GitOps 在运行过程中以 Git 为声明性基础设施和应用的单一事实来源。
GitOps 使用 Git PR 来自动管理基础设施的置备和部署。Git  Repo 包含系统的全部状态，因此系统状态的修改痕迹既可查看也可审计。
GitOps 围绕开发者经验而构建，可帮助团队使用与软件开发相同的工具和流程来管理基础架构。除了 Git 以外，GitOps 还可以按照自己的需求选择工具。
一句话总结：GitOps 就是使用 Git 拉取请求来验证和自动部署系统基础设施变更的实践。

## 工作流

我们可以认为 GitOps 是基础架构即代码（IaC）的一个演变，以 Git 为基础架构配置的版本控制系统。IaC 通常采用声明式方法来管理基础架构，定义系统的期望状态，并跟踪系统的实际状态。
与 IaC 一样，GitOps 也要求声明式描述系统的期望状态。通过使用声明式工具，所有的配置文件和源代码都可以在 Git 中进行版本控制。
1. CI/CD 流程通常由外部事件触发，比如代码被推送到了存储库中。在 GitOps 工作流中，要进行变更，需要通过 PR 来修改 Git 仓库的状态。   
2. 一旦这些 PR 被批准和合并，它们将自动应用于实时基础设施。开发人员可以使用他们的标准工作流以及 CI/CD 流水线。 
3. 同时搭配使用 Kubernetes 与 GitOps 时，Operator 通常为 Kubernetes Operator。
4. Operator 会将 Repo 中的期望状态与所部署基础设施的实际状态进行比较。每当注意到实际状态与存储库中的期望状态存在差异时，Operator 便会更新基础设施。Operator 还可以监控容器镜像仓库，并以同样的方式进行更新，从而部署新的镜像。

GitOps 中还有一个重要概念——可观察性，指可观察的任何系统状态。GitOps 的可观察性有助于确保观察到的状态（或实际状态）与理想状态一致。 
使用 PR 和 Git 之类的版本控制系统让部署过程清晰可见。这样一来，便可查看和跟踪对系统进行的任何变更，提供审计跟踪，并在出现问题时进行回滚变更。
GitOps 工作流可以提高生产率，加快开发和部署速度，同时提高系统的稳定性和可靠性。

## 部署策略

GitOps 的部署策略有两种实现方式：基于推送（Push）和基于拉取（Pull）的部署。两种部署类型之间的区别在于如何确保部署环境实际上类似于所需的基础架构。在可能的情况下，应该首选基于 Pull 的方法，因为它被认为更安全，因此是实施 GitOps 的更好实践。

### 基于 Push

基于 Push 的部署策略由流行的 CI/CD 工具实现，例如 Jenkins、CircleCI 或 Travis CI。应用程序的源代码与部署应用程序所需的配置一起位于应用程序代码库中。每当更新应用程序代码时，都会触发构建管道，构建容器镜像，最后使用新的部署描述符更新环境配置仓库。

![](/posts/devops-vs-gitops/push-based-gitops.png)

### 基于 Pull

基于 Pull 的部署策略使用与基于 Push 的基本相同的概念，但在部署流水线的工作方式上有所不同。传统的 CI/CD 管道由外部事件触发，例如当新代码被推送到应用程序存储库时。使用基于 Pull 的部署方法，引入了 Operator。它通过不断地将环境存储库中的所需状态与已部署基础架构中的实际状态进行比较，接管 Pipeline 的角色。每当发现差异时，Operator 都会更新基础架构以匹配环境存储库。此外，可以监控镜像仓库以查找要部署的镜像版本。

![](/posts/devops-vs-gitops/pull-based-gitops.png)

# 小结

|	|DevOps|GitOps|
|---|---|---|
|定位|文化，专注于 CI/CD 的文化|技术，使用 Git 管理基础设施配置和软件部署的技术|
|依赖|CI/CD Pipeline|Git|
|协作|云配置即代码和供应链管理|Kubernetes、IaC 和各种 CI/CD Pipeline|
|目标|降低错误率，消除团队中的孤岛，减少成本工作等|速度、准确性、清洁代码、提高生产力|
|正确性|不太关注代码的精度|高度关注准确和简洁的代码|
|灵活性|不那么严格，更开放|更严格，更少开放|

# 参考资料
- [DevOps in a Nutshell – What Is DevOps, Really?](https://www.bunnyshell.com/blog/what-is-devops)
- [Understanding DevOps](https://www.redhat.com/zh/topics/devops)
- [What is GitOps?](https://www.redhat.com/zh/topics/devops/what-is-gitops)
- [Guide To GitOps](https://www.weave.works/technologies/gitops/)
- [GitOps vs DevOps: Differences and Why They are Better Together](https://www.aquasec.com/cloud-native-academy/devsecops/gitops-vs-devops/)
- [GitOps vs DevOps – What’s the difference?](https://eleks.com/blog/gitops-vs-devops/)
