---
date: 2022-12-16
title: Terraform 执行状态和阶段关系
tag: [Terraform]
categories: [IaC]
---

Terraform 每次运行都经过多个操作阶段（Pending、Plan、Cost Estimation、Policy Check、Apply 和 Completion），Terraform Cloud 将这些阶段的进度显示为运行状态。
在 Terraform Cloud 主页上的工作区列表中，每个工作区都显示了它当前正在处理的运行状态。（或者，如果没有正在进行的运行，则为最近完成的运行的状态。）

<!--more-->

## Pending

阶段状态：
- Pending：Terraform Cloud 尚未开始运行操作。Terraform Cloud 按照排队的顺序处理每个工作区的运行，并且运行在每次运行完成之前一直处于挂起状态。

离开状态：
- Discarded：如果用户在开始前放弃运行，则运行不会继续。
- Planning：如果运行在队列中排在首位，它会自动进入 Plan 阶段。

图示：

<img src="/iac/pending.png" width="300" height="200" style="display: block; margin: auto;">

## Fetching

Terraform Cloud 可能需要在开始计划之前从 VCS 获取配置。当所有运行完成后，Terraform Cloud 会自动存档通过 VCS 创建的配置版本，然后重新获取文件以供后续运行使用。

阶段状态：
- Fetching：如果 Terraform Cloud 尚未从 VCS 获取配置，则运行将进入此状态，直到配置可用。

离开状态：
- Plan Errored：如果 Terraform Cloud 在从 VCS 获取配置时遇到错误，则运行不会继续。
- 如果 Terraform 成功获取配置，则运行进入下一阶段。

图示：

<img src="/iac/fetching.png" width="300" height="200" style="display: block; margin: auto;">

## Pre-plan（可选）

创建 [Run Task](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-tasks)，并关联 Workspace，选择在 “Pre-plan” 时执行，才有 Pre-plan 阶段。在此阶段，Terraform Cloud 将有关运行的信息发送到已配置的外部系统，并等待 passed 或 failed 响应以确定运行是否可以继续。

阶段状态：
- Pre-planning： Terraform Cloud 正在等待配置的外部系统的响应。
  - 外部系统必须首先响应确认 200 OK请求正在进行中。之后，他们有 10 分钟的时间返回状态 passed、running或 failed。如果超时到期，Terraform Cloud 假定运行任务处于状态 failed。

离开状态：
- Plan Errored：如果任何强制性任务失败，运行将跳至完成（Plan Errored状态）。
- Planning：如果任何咨询任务失败，运行将进入计划状态，并带有关于失败任务的可见警告。
- Plan Errored/Planning：如果单次运行包含强制任务和建议任务的组合，Terraform 会采取最严格的操作。例如，如果有两项建议任务成功而一项强制任务失败，则运行失败。
- Canceled：如果用户取消了运行，运行将以 Canceled 状态结束。

图示：

<img src="/iac/pre-plan.png" width="600" height="800" style="display: block; margin: auto;">

场景：
- Blink：实现无代码自动化，以跨云工具和服务管理 IaC 工作流
- [Check Point](https://get.spectralops.io/docs/integration/tfc/)：在规划阶段之前检测安全配置错误
- [Tines](https://www.tines.com/blog/handle-terraform-cloud-infrastructure-requests-automatically)：帮助用户批准和记录来自 Terraform Cloud 的基础设施请求
- [Torq](https://docs.torq.io/integrations/hashicorp-terraform)：为基础设施扫描、批准生命周期和使用 Jira 等票务系统进行跟踪创建自动化的无代码工作流

## Plan

根据 Terraform Cloud 是否需要从 VCS 获取配置，一次运行在计划阶段会经历不同的步骤。当所有运行完成后，Terraform Cloud 会自动存档通过 VCS 创建的配置版本，然后重新获取文件以供后续运行使用。

阶段状态：
- Planning： Terraform Cloud 当前正在运行 terraform plan。
- Needs Confirmation：`terraform plan 已完成。运行有时会在此状态下暂停，具体取决于工作区和组织设置。

离开状态：
- Plan Errored：如果 terraform plan命令失败，则运行不会继续。
- Canceled：如果用户通过按下“取消运行”按钮取消了计划，则运行不会继续。
- Planned and Finished：如果计划成功且没有任何更改，并且既不会进行成本估算，也不会进行 Sentinel 策略检查，则 Terraform Cloud 认为运行已完成。
- 如果计划成功并需要更改：
  - Cost Estimation：如果启用了成本估算，则运行会自动进入成本估算阶段。
  - Sentinel Policy Check：如果禁用成本估算并启用 Sentinel 策略，则运行会自动进入策略检查阶段。
  - Apply：如果没有 Sentinel 策略并且可以自动应用计划，则运行会自动进入应用阶段。如果在工作区上启用了 auto-apply ，并且计划由新的 VCS 提交或具有应用运行权限的用户排队，则可以自动应用计划。
  - 如果没有 Sentinel 策略并且 Terraform Cloud 无法自动应用该计划，则运行将暂停在 Needs Confirmation 状态，直到有权应用运行的用户采取行动。如果授权用户批准申请，则运行将进入 Apply阶段。如果授权用户拒绝申请，则运行不会继续（Discarded 状态）。

图示：

<img src="/iac/plan.png" width="500" height="500" style="display: block; margin: auto;">

## Post-plan（可选）

创建 [Run Task](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-tasks)，并关联 Workspace，选择在 “Post-plan” 时执行，才有 Post-plan 阶段。在此阶段，Terraform Cloud 将有关运行的信息发送到已配置的外部系统，并等待 passed 或 failed 响应以确定运行是否可以继续。

阶段状态：
- Post-planning： Terraform Cloud 正在等待配置的外部系统的响应。
  - 外部系统必须首先响应确认 200 OK请求正在进行中。之后，他们有 10 分钟的时间返回状态 passed、running或 failed。如果超时到期，Terraform Cloud 假定运行任务处于状态 failed。
离开状态：
- Plan Errored：如果任何强制性任务失败，运行将跳至完成 Plan Errored 状态。
- Applying：如果任何咨询任务失败，运行将进入 Applying 状态，并带有关于失败任务的可见警告。
- Plan Errored/Applying：如果单次运行包含强制任务和建议任务的组合，Terraform 会采取最严格的操作。例如，如果有两项建议任务成功而一项强制任务失败，则运行失败。如果一项强制性任务成功而两项建议性任务失败，则运行成功并发出警告。
- Canceled：如果用户取消了运行，运行将以 Canceled 状态结束。

图示：

<img src="/iac/pending.png" width="300" height="200" style="display: block; margin: auto;">

场景：

用户期望在运行生命周期的各个阶段能做更多的事情，Post-plan 普遍的用例是确保团队遵守组织的安全性和合规性要求：
- 安全性：确保您没有使用 Snyk、Bridgecrew、Tenable、Moderne、Sophos、Aqua Security、Firefly 和 Lightlytics 等工具提供会导致安全问题的错误配置
- 成本控制：在使用 Infracost、Vantage 或 Kion 进行任何更改之前提供对基础设施成本的可见性
- 法规遵从性：使用 Bridgecrew 或 Kion 确保遵守各种法规，例如 HIPAA、GDPR 或 PCI-DSS

## OPA Policy Check（可选）

此阶段仅在启用[开放策略代理 (OPA) 策略](https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/opa)并在 terraform plan 成功之后和成本估算之前运行时才会发生。在此阶段，Terraform Cloud 检查计划是否符合工作空间的 OPA 策略集中的策略。

阶段状态：

- Policy Check：Terraform Cloud 正在根据 OPA 政策集检查计划。
- Policy Override：策略检查已完成，但强制策略失败。运行暂停，除非用户手动覆盖策略检查失败，否则 Terraform 无法执行应用。
- Policy Checked：策略检查成功，Terraform 可以应用该计划。如果工作区未设置为自动应用运行，运行可能会在此状态下暂停。

离开状态：

如果任何强制性 Policy 失败，运行将暂停在 Policy Override 状态。运行完成以下工作流程之一：
- Discarded：当具有应用运行权限的用户放弃运行时，运行停止并进入 Discarded 状态。
- Policy Checked：当有权管理策略覆盖的用户覆盖失败的策略时，运行会进入 Policy Checked 状态。Policy Checked 状态意味着没有强制性策略失败或用户执行了手动覆盖。
一旦运行达到 Policy Checked 状态，运行将完成以下工作流程之一：
- 如果 Terraform 可以自动应用计划，则运行将进入 Apply 阶段。自动应用要求在工作区启用 auto-apply。
- 如果 Terraform 无法自动应用该计划，则运行将暂停在 Policy Checked 状态，直到有权应用运行的用户采取行动。如果用户批准申请，则运行进入 Apply 阶段。如果用户拒绝申请，则运行停止并进入 Discarded 状态。

图示：

<img src="/iac/opa.png" width="500" height="500" style="display: block; margin: auto;">

## Cost Estimation（可选）

此阶段仅在启用成本估算时发生。成功后 terraform plan，Terraform Cloud 使用计划数据来估算计划中找到的每个资源的成本。

阶段状态：
- Cost Estimating： Terraform Cloud 目前正在估算计划中的资源。
- Cost Estimated：已完成成本估算。

离开状态：
- 如果成本估算成功或出错，则运行进入下一阶段。
- Planned and Finished：如果没有策略检查或应用，则运行不会继续。

图示：

<img src="/iac/cost.png" width="500" height="500" style="display: block; margin: auto;">

## Sentinel Policy Check（可选）

terraform plan 执行成功后，Terraform Cloud 会检查计划是否符合策略，以确定是否可以应用。

阶段状态：
- Policy Check：Terraform Cloud 当前正在根据组织的政策检查计划。
- Policy Override：策略检查已完成，但软强制性策略失败，因此未经有权为组织管理策略覆盖的用户批准，应用无法继续。运行在此状态下暂停。
- Policy Checked：策略检查成功，Sentinel 将允许申请继续进行。运行有时会在此状态下暂停，具体取决于工作区设置。

离开状态：
- Plan Errored：如果任何硬性强制策略失败，则运行不会继续）。
- Policy Override：如果任何软强制策略失败，运行将暂停在 Policy Override 状态。
  - Policy Checked：如果有权管理策略覆盖的用户覆盖失败的策略，则运行会进入 Policy Checked 状态。
  - Discarded：如果有权应用运行的用户放弃运行，则运行不会继续。
- 如果运行达到 Policy Checked 状态（没有强制策略失败，或者软强制策略被覆盖）：
  - 如果计划可以自动应用，则运行会自动进入应用阶段。如果在工作区上启用了 auto-apply 设置，并且计划由新的 VCS 提交或具有应用运行权限的用户排队，则可以自动应用计划。
  - 如果无法自动应用该计划，则运行将暂停在 Policy Checked 状态，直到有权应用运行的用户采取行动。如果他们批准申请，则运行会进入申请阶段，如果他们拒绝申请，则不会继续（Discarded）。

图示：

<img src="/iac/policy-check.png" width="500" height="500" style="display: block; margin: auto;">

## Pre-Apply（可选）

创建 [Run Task](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-tasks)，并关联 Workspace，选择在 “Pre-apply” 时执行，才有 Pre-apply 阶段。在此阶段，Terraform Cloud 将有关运行的信息发送到已配置的外部系统，并等待 passed 或 failed 响应以确定运行是否可以继续。

阶段状态：
- Pre-apply running：Terraform Cloud 正在等待配置的外部系统的响应。
  - 外部系统必须首先响应确认 200 OK请求正在进行中。之后，他们有 10 分钟的时间返回状态 passed、running或 failed。如果超时到期，Terraform Cloud 假定运行任务处于状态 failed。

离开状态：
- Apply Errored：如果任何强制性任务失败，运行将跳至完成。
- Applying：如果任何咨询任务失败，运行将进入 Applying 状态，并带有关于失败任务的可见警告。
- Apply Errored/Applying：如果单次运行包含强制任务和建议任务的组合，Terraform 会采取最严格的操作。例如，如果有两项建议任务成功而一项强制任务失败，则运行失败。
- Canceled：如果用户取消了运行，运行将以 Canceled 状态结束。

图示：

<img src="/iac/pre-apply.png" width="500" height="500" style="display: block; margin: auto;">

场景：

基础设施的日常运维中，检查 Terraform 配置是重要的一步。通常，团队会采用审查和批准流程，这通常会导致计划最初生成和应用之间出现延迟。这段时间，基础架构可以更改，可以强制执行，并且可以添加新的合规性规则，在同一个 Terraform Run 中执行。

## Apply

阶段状态：
- Applying：Terraform Cloud 当前正在执行 terraform apply。

离开状态：
- Applied：如果应用成功，则以 Applied 状态结束。
- Apply Errored：如果应用失败，则以 Apply Errored 状态结束。
- Canceled：如果用户通过按 Cancel Run 取消应用，则以 Canceled 状态结束。

图示：

<img src="/iac/apply.png" width="500" height="400" style="display: block; margin: auto;">


## Completion

如果 Apply 完成，或者任何一步失败，或者无事可做，或者用户选择不继续，则一次运行结束。一旦运行完成，队列中的下一个运行可以进入计划阶段。

阶段状态：
- Applied：一次运行成功应用。
- Planned and Finished: terraform plan的输出已经与当前的基础设施状态相匹配，所以 terraform apply不需要做任何事情。
- Apply Errored：命令 terraform apply失败，可能是由于缺少或配置错误的提供程序或对提供程序的非法操作。
- Plan Errored：命令 terraform plan失败（通常需要修复变量或代码），或者硬性强制性 Sentinel 策略失败。无法应用运行。
- Discarded：用户选择不继续此运行。
- Canceled：用户使用“取消运行”按钮中断了 terraform plan或 terraform apply命令。

## Reference
- https://developer.hashicorp.com/terraform/cloud-docs/run/states
- https://www.hashicorp.com/blog/pre-plan-apply-run-tasks-now-available-in-terraform-cloud
- https://www.hashicorp.com/blog/terraform-cloud-run-tasks-are-now-generally-available