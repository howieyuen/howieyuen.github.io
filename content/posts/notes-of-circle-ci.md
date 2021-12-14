---
title: Circle CI 学习笔记
date: 2021-12-14
tags: [CircleCI, Orb]
categories: [CI/CD]
---

# 1. 基本概念

首先介绍下 Circle CI 的[基本概念](https://circleci.com/docs/2.0/concepts/)，帮助大家理解 Circle CI 是如何管理 CI/CD 流程。

## 1.1 管控层

- Project：Circle CI 项目在你的 VCS（GitHub 或者 Bitbucket）中 ，共享代码库的名称。在 Circle CI 的主页中点击 _Projects_ 可以自由添加项目进入仪表盘，以追踪代码变更。

- Configuration：Circle CI 遵循配置即代码（Configuration as Code）原则，整个 CI/CD 的流程都是通过 _config.yml_ 文件，该文件位于项目根目录下的 _.circleci_ 文件夹中。以下按照术语和依赖性顺序，列举了 Circle CI 最常见的组件：
  - Pipeline：代码表整个配置，仅用于 Circle CI 云
  - Workflow：负责编排多个 Job
  - Job：负责运行执行命令的一系列 Step
  - Step：运行命令（例如安装依赖项或运行测试）和 shell 脚本
  
- User Type：大多数都是从 VCS 帐号中继承的权限，主要包括以下四种：

  - 组织管理员（Organization Administrator）：从 VCS 继承的权限级别。
    - GitHub：Owner 权限，并且至少关注了一个在 Circle CI 上的构建的项目。
    - Bitbucket：Admin 权限，并且至少关注了一个在 Circle CI 上的构建的项目。
- 项目管理员（Project Administrator）：是将 GitHub 或 Bitbucket 的 Repo 作为项目添加到 Circle CI 的用户。
  - 用户（User）：是组织内的个人用户，从 VCS 继承。
- Circle CI 用户（Circle CI User）：可以使用用户名和密码登录 Circle CI 平台的任何人。
  

## 1.2 配置层

### ~~1.2.1 Pipeline~~

管道是触发项目工作时运行的全套流程，它包含了 Workflow，Workflow 会协调 Job，这些都应在项目的配置文件中。**Pipeline 在 Circle CI 2.X 版本不可用**。

### 1.2.2 Orb

[Orb](https://circleci.com/docs/2.0/orb-intro/) 是可重复使用的代码片段，有助于自动化重复流程、加快项目设置并使其易于与第三方工具集成。

```yaml
version: 2.1

orbs:
	maven: circleci/maven@0.0.12

workflow:
	maven_test:
		jobs:
			- maven/test
```

### 1.2.3 Job

作业是配置的基础，也是 Step 的集合，根据需要运行命令/脚本。每个作业必须声明 Executor，它可以是：

- docker：必须指定镜像
- machine：存在默认镜像
- windows：必须使用 Window Orb
- macos：必须执行 XCode 版本

### 1.2.4 Executor 和 images

每个作业都可以在唯一的执行器中运行，可以是 Docker 容器，也可以是 Linux/Windows 的虚拟机。**Circle CI 2.X 不支持 MacOS**。

```yaml
version: 2.1
jobs:
 build1: # job 名称
   docker: # 使用 docker 环境
     - image: buildpack-deps:trusty # 主镜像
       auth:
         username: mydockerhub-user
         password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
     - image: postgres:9.4.1 # 次镜像
       auth:
         username: mydockerhub-user
         password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
       # 次容器可以通过 localhost 访问主容器
       environment: # 指定环境变量
         POSTGRES_USER: root
 build2:
   machine: # 使用 Linux 环境
     image: ubuntu-2004:202010-01 # 指定 Linux VM 版本 
 build3:
   macos: # 使用 macOS 环境
     xcode: "11.3.0" # 指定 Xcode 版本
```


### 1.2.5 Step

步骤通常是一组可执行命令，例如内置命令 `checkout` 通过 SSH 检查源码；`run` 可以自定义命令。命令可以定义成全局，供多次使用。

```yaml
jobs:
  build:
    docker:
      - image: <image-name-tag>
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
    steps:
      - checkout # 内置 step，checkout 代码
      - run: # 内置 step，用于执行命令
          name: Running tests # step 名称
          command: make test # 默认情况下，可执行命令在非登录 shell 中使用 /bin/bash -eo pipefail 选项运行。
```

### 1.2.6 Image

镜像就是一个打包的系统，在 _.circleci/config.yml_ 中定义的主镜像，这是 Docker 或者其他执行器执行 Job 命令的地方。

### 1.2.7 Workflow

工作流定义了 Job 列表及其运行顺序。可以并行、串行、按计划或使用 approval 的来运行 Job。

```yaml
#...
workflows:
  build_and_test: # workflow 名称
    jobs:
    	- build
      - test: # test 依赖 build 执行成功 
          requires: 
            - build
      - hold: # hold 依赖 build 和 test 执行成功，并且需要 approve
          type: approval
          requires:
            - build
            - test
      - deploy: # deploy 依赖 hold 执行成功 
          requires:
            - hold
```

### 1.2.8 Cache/Workspace/Artifact

- Cache 在对象存储中保存的是文件或文件目录，例如依赖项或源代码。每个作业都可能包含特殊步骤，例如使用先前作业缓存的依赖项来加速构建。

```yaml
version: 2.1

jobs:
  build1:
    docker:
      - image: <image-name-tag>
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
    steps:
      - checkout
      - save_cache: # 内置 step，cache 依赖环境变量提供的 key
          key: v1-repo-{{ .Environment.CIRCLE_SHA1 }}
          paths:
            - ~/circleci-demo-workflows

	# build2 使用 build1 的缓存
  build2:
    docker:
      - image: <image-name-tag>
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
    steps:
      - restore_cache: # 内置 step，恢复缓存依赖项
          key: v1-repo-{{ .Environment.CIRCLE_SHA1 }}
```

- Workspace 是一种感知 Workflow 的存储机制。Workspace 保存 Job 独有的数据，下游 Job 可能需要这些数据。每个 Workflow 都有一个与之关联的临时 Workspace。Workspace 可用于将同一 Workflow 下某个 Job 在执行期间构建的特殊数据传递给其他 Job。
- Artifact 在 Workflow 完成后保留数据，可用于构建并输出长期存储。

```yaml
version: 2.1

jobs:
  build1:
#...
    steps:
      - persist_to_workspace: # 内置 step
      # 将指定路径(workspace/echo-output) 保留到 Workspace 中以供下游 Job 使用
      # 路径必须是绝对路径，或来自 working_directory 的相对路径。
      # 这是容器里的目录，是工作空间的根目录。
          root: workspace
            # 必须是根目录的相对路径
          paths:
            - echo-output

  build2:
#...
    steps:
      - attach_workspace: # 内置 step
      		# 路径必须是绝对路径，或来自 working_directory 的相对路径。
          at: /tmp/workspace
          
  build3:
#...
    steps:
      - store_artifacts: # 内置 step
          path: /tmp/artifact-1
          destination: artifact-file
```

## 1.3 小结

走到这里，应该对 Circle CI 有了粗浅的认识，但想要上手甚至是玩转，还需要更多指导和实践。这是初次尝试 Circle CI 时写 [config.yml](https://github.com/howieyuen/hello-circle-ci/blob/circleci-project-setup/.circleci/config.yml)。当然 Circle CI 提供了丰富的配置项，详情可以移步这里：[Circle CI 配置参考](https://circleci.com/docs/2.0/configuration-reference/)。



# 2. Orb 简介

Circle CI [Orb](https://circleci.com/docs/2.0/orb-concepts/) 是可共享的配置包，可以看做是 lib 库，选择合适的 Orb ，会让 Circle CI 配置编写更容易。Orb 中可配置元素主要包括：Command、Executor 和 Job。

## 2.1 Command 示例

```yaml
version: 2.1

orbs:
  aws-s3: circleci/aws-s3@x.y.z

jobs:
  build:
    docker:
      - image: 'cimg/python:3.6'
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
    steps:
      - checkout
      - run: mkdir bucket && echo "lorem ipsum" > bucket/build_asset.txt
      # 使用orb：aws-s3 的 copy 命令
      - aws-s3/copy:
          from: bucket/build_asset.txt
          to: 's3://my-s3-bucket-name'
```

## 2.2 Executor 示例

```yaml
description: >
  Select the version of NodeJS to use. Uses CircleCI's highly cached convenience
  images built for CI.
docker:
  - image: 'cimg/node:<<parameters.tag>>'
    auth:
      username: mydockerhub-user
      password: $DOCKERHUB_PASSWORD  # context/project UI 环境变量引用
parameters:
  tag:
    default: '13.11'
    description: >
      Pick a specific cimg/node image version tag:
      https://hub.docker.com/r/cimg/node
    type: string
```

## 2.3 Job 示例

```yaml
version: 2.1

orbs:
  <orb>: <namespace>/<orb>@x.y #orb 版本

workflows:
  use-orb-job:
    jobs:
      - <orb>/<job-name>
```



# 3. Orb 开发实践

Orb 实践，简单来说包括这些阶段：取名、分类、说明、context 限制、发布、升级。在实践之前，先安装 Orb 开发套件：[*circleci*](https://circleci.com/docs/2.0/local-cli/)。

## 3.1 取名

一个好的 Orb 命名，是由 Namespace + 名称组成，由正斜杠分隔。Namespace 代表拥有和维护 Orb 的个人、公司或组织，而 Orb 名称本身应描述提供的产品、服务或操作。取名之前，先注册 Namespace，这里需要关联 Circle CI 帐号，所以需要获取 [Access Token](https://app.circleci.com/settings/user/tokens)（注意保存），然后就可以使用[注册 Namespace](https://circleci.com/docs/2.0/orb-author-intro/#register-a-namespace) 命令 `circleci namespace create <name> <vcs-type> <org-name> [flags]`，详细如下：

```bash
➜  ~ circleci namespace create howieyuen-orb github howieyuen --token 5b8b950571bbe70ad5fa15e22ec052b52702f241
You are creating a namespace called "howieyuen-orb".

This is the only namespace permitted for your github organization, howieyuen.

To change the namespace, you will have to contact CircleCI customer support.

? Are you sure you wish to create the namespace: `howieyuen-orb` Yes
Namespace `howieyuen-orb` created.
Please note that any orbs you publish in this namespace are open orbs and are world-readable.
```

## 3.2 初始化

*circleci* 工具提供了[初始化 Orb](https://circleci.com/docs/2.0/orb-author/#orb-development-kit) 命令：`circleci orb init`。它包含了分类、说明等操作。但有几个前置操作需要完成：

1. 设置 [Context](https://circleci.com/docs/2.0/contexts/#restricting-a-context)：Organization Settings > Contexts
1. [启用 Uncertified Orbs](https://circleci.com/docs/2.0/orbs-faq/#using-uncertified-orbs)：Organization Settings > Security
1. 新建 GitHub Repo：https://github.com/howieyuen/hello-circleci-orb

到此，就可以执行初始化命令 `circleci orb init <path> [flags]`，详细如下：

```bash
➜ circleci orb init hello-circleci-orb --token 5b8b950571bbe70ad5fa15e22ec052b52702f241
Note: This command is in preview. Please report any bugs! https://github.com/CircleCI-Public/circleci-cli/issues/new/choose
? Would you like to perform an automated setup of this orb? Yes, walk me through the process.
Downloading Orb Project Template into hello-circleci-orb
A few questions to get you up and running.
? Are you using GitHub or Bitbucket? GitHub
? Enter your github username or organization howieyuen
? Enter the namespace to use for this orb howieyuen-orb
Saving namespace howieyuen-orb as default
? Orb name hello-circleci-orb
? What categories will this orb belong to? Testing
? Automatically set up a publishing context for your orb? Yes, set up a publishing context with my API key.
`orb-publishing` context already exists, continuing on
? Would you like to set up your git project? Yes
? Enter your primary git branch. main
? Enter the remote git repository git@github.com:howieyuen/hello-circleci-orb.git
Thank you! Setting up your orb...
An initial commit has been created - please run 'git push origin main' to publish your first commit!
? I have pushed to my git repository using the above command Yes
Your orb project is building here: https://circleci.com/gh/howieyuen/hello-circleci-orb
You are now working in the alpha branch.
Once the first public version is published, you'll be able to see it here: https://circleci.com/developer/orbs/orb/howieyuen-orb/hello-circleci-orb
View orb publishing doc: https://circleci.com/docs/2.0/orb-author
```

该步骤完成后，项目默认位于 alpha 分支，该分支代码是从模板克隆，模板详见[链接](https://github.com/CircleCI-Public/Orb-Project-Template)。

## 3.3 发布

在发布之前，可以将 alpha 分支，推送到远端，触发 _test-pack workflow_，执行基本验证、lint 和单元测试。

目前为止，Orb 只是在 dev 阶段，其他用户无法从 [Orb 市场](https://circleci.com/developer/orbs)检索到。正式发布的流程也很简单，是根据 Orb 仓库的 Commit Message 的前缀字段 `[semver:<increment>]`，来确定是否需要发布，以及发布的版本号。Orb 发布也遵循[语义化版本](https://circleci.com/docs/2.0/orb-concepts/#semantic-versioning)控制：

| Increment | 描述                |
| --------- | ------------------- |
| major     | 发布 1.0.0 增量版本 |
| minor     | 发布 x.1.0 增量版本 |
| patch     | 发布 xx1 增量版本   |
| skip      | 不要发布版本        |

因此，发布版本，只需要修改 alpha 分支的 commit message，将 “skip” 改成 *increment* 任何一个值即可，Circle CI 就会触发 _integration_test_deploy_ 工作流，待完成后，即可在  [Orb 市场](https://circleci.com/developer/orbs)检索到刚才发布的 Orb：[howieyuen-orb/hello-circleci-orb@1.0.0](https://circleci.com/developer/orbs/orb/howieyuen-orb/hello-circleci-orb)。

# 4. 参考资料

- [Circle CI 基本概念]([https://circleci.com/docs/2.0/concepts/](https://circleci.com/docs/2.0/concepts/#))
- [Circle CI 配置样例](https://circleci.com/docs/2.0/sample-config)
- [Circle CI 配置参考](https://circleci.com/docs/2.0/configuration-reference/#)
- [Circle CI 可重用配置指南](https://circleci.com/docs/2.0/reusing-config/)
- [Orb 介绍](https://circleci.com/docs/2.0/orb-intro/)
-  [Orb 概念](https://circleci.com/docs/2.0/orb-concepts/)
-  [CircleCI 本地 CLI](https://circleci.com/docs/2.0/local-cli/)