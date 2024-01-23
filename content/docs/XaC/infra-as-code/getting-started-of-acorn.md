---
date: 2022-12-16
title: Acorn：k8s 应用部署框架
tags: [Acorn]
categories: [IaC]
---

## 概述

Acorn 是一个应用程序打包和部署框架，可简化在 Kubernetes 上运行的应用程序。 Acorn 能够将应用程序的所有 Docker 镜像、配置和部署规范打包到单个 Acorn 镜像工件中。 此工件可发布到任何 OCI 容器注册表，允许它部署在任何开发、测试或生产环境的 Kubernetes 上。 Acorn 镜像的可移植性使开发人员能够在本地开发应用程序并转移到生产环境，而无需切换工具或技术堆栈。

开发人员通过在 Acornfile 中描述应用程序配置来创建 Acorn 镜像。 Acornfile 描述了整个应用程序，没有 Kubernetes YAML 文件的所有样板。 Acorn CLI 用于在任何 Kubernetes 集群上构建、部署和操作 Acorn 镜像。

## 架构

- acorn：安装在终端用户的机器上，并针对在 Kubernetes 集群中运行的 acorn-apiserver 交互
- acorn-apiserver：k8s 风格的 API 服务器，通过 k8s aggregation layer 访问
- acorn-controller：负责将 Acorn 应用程序转换为实际的 Kubernetes 资源
- buildkit & internal registry：镜像构建服务、Buildkit 和内部镜像注册表作为同级容器部署在单个 pod 中。当 Buildkit 构建新的 Acorn 镜像时，这简化了两个组件之间的通信

![](/iac/acorn.png)

## 工作流

下图说明了用户在使用 Acorn 时所采取的步骤：

![](/iac/workflow.png)

1. 用户创建一个 Acornfile 来描述应用程序。
2. 用户调用 Acorn CLI 从 Acornfile 构建一个 Acorn 镜像。
3. Acorn 镜像被推送到 OCI 注册表。
4. 用户调用 Acorn CLI 将 Acorn 镜像部署到 Acorn 运行时，它可以安装在任何 Kubernetes 集群上。

## Acron vs Helm

1. Helm chart 是 Kubernetes YAML 文件的模板，而 Acornfiles 定义应用程序级结构。Acorn 是 Kubernetes 之上的抽象层。Acorn 用户不直接使用 Kubernetes YAML 文件。按照设计，使用 Acorn 不需要 Kubernetes 知识。
2. Helm 用户可以将任何 Kubernetes 工作负载打包到 Helm chart 中，而 Acorn 旨在打包应用程序而不是系统级驱动程序、插件和代理。Acorn 支持任何类型的应用程序，无状态和有状态。应用程序在它们自己的命名空间中运行。应用程序不需要特权容器。应用程序运行在 Kubernetes 上，但不调用底层 Kubernetes API 或通过定义自定义资源将底层 etcd 作为数据库使用。
3. Acornfiles 定义应用程序级结构，例如 Docker 容器、应用程序配置和应用程序部署规范。Acorn 为 Kubernetes 上的应用程序部署带来了结构。这与在 Helm chart 中不受限制地使用 Kubernetes YAML 文件形成鲜明对比。

## Acornfile

Acornfile 的主要目标是快速轻松地描述如何构建、开发和运行容器化应用程序。该语法与熟悉的 JSON 和 YAML 非常相似。

由 Acornfile 定义，acorn build 生成的结果是应用程序的完整包。它在单个 OCI 镜像中包含所有容器镜像、机密、数据、嵌套的 Acorns 等，可以通过注册表进行分发。

```js
args: {
  // Configure your personal welcome text
  welcome: "Hello Acorn User!!"
}

containers: {
  app: {
    build: "."
    env: {
      "PG_USER": "postgres"
      "PG_PASS": "secret://quickstart-pg-pass/token"
      "WELCOME": args.welcome
      if args.dev { "FLASK_ENV": "development" }
    }
    dependsOn: [
      "db",
      "cache"
    ]
    if args.dev { dirs: "/app": "./" }
    ports: publish: "5000/http"
  }
  cache: {
    image: "redis:alpine"
    ports: "6379/tcp"
  }
  db: {
    image: "postgres:alpine"
    env: {
      "POSTGRES_DB": "acorn"
      "POSTGRES_PASSWORD": "secret://quickstart-pg-pass/token"
    }
    dirs: {
      if !args.dev {
        "/var/lib/postgresql/data": "volume://pgdata?subpath=data"
      }
    }
    files: {
      "/docker-entrypoint-initdb.d/00-init.sql": "CREATE TABLE squirrel_food (food text);"
      "/docker-entrypoint-initdb.d/01-food.sql": std.join([for food in localData.food {"INSERT INTO squirrel_food VALUES ('\(food)');"}], "\n")
    }
    ports: "5432/tcp"
  }
}

localData: {
  food: [
    "acorns",
    "hazelnuts",
    "walnuts"
  ]
}

volumes: {
  if !args.dev {
    "pgdata": {
      accessModes: "readWriteOnce"
    }
  }
}

secrets: {
  "quickstart-pg-pass": {
      type: "token"
  }
}
```

## References

- https://github.com/acorn-io/acorn
- https://docs.acorn.io/reference/acornfile