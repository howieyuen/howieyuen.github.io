---
title: 如何管理 GitOps Secret：详细指南
date: 2022-09-01
categories: [GitOps, Secret]
---

GitOps 正变得越来越流行。越来越多的公司开始使用 Git 作为其基础设施和应用程序配置的真实来源。然而，伴随其优势而来的是挑战。例如，如果你的所有配置都存储在 Git 中，你如何管理 Secret？你不能简单地将密码和令牌以明文形式提交到 Git 存储库，即使该存储库是私有的并且只有少数人可以访问它。在这篇文章中，你将学习如何安全地管理 GitOps Secret。敬请关注。

<!--more-->

> 原文链接：[How to Manage GitOps Secrets: A Detailed Guide](https://releasehub.com/blog/how-to-manage-gitops-secrets-a-detailed-guide)

## GitOps vs Secret

如果你以前从未使用过 GitOps，这里有一个简短的介绍。GitOps 是一种纯粹通过 Git 存储库以声明方式管理基础设施和应用程序配置的方法。它的工作原理如下：你将所有配置存储在 Git 中，然后在某处安装一个 GitOps 工具，该工具会持续监控对该 Git 存储库的更改，并在检测到存储库中发生更改时应用基础架构和应用程序更改。GitOps 的全部意义在于，你的所有基础架构和应用程序配置都拥有一个集中的、单一的事实点。GitOps 最常与 Kubernetes 一起使用。

但正如本文开头所述，使用 GitOps 时存在一些挑战。而最大的一个是 Secret 管理。你的基础设施将需要许多 Secret。你的应用程序配置可能也充满了 Secret。不用说，以纯文本形式将 Secret 存储在 Git 存储库中是一个安全漏洞。即使该存储库是私有的也是如此。你需要一个不同的解决方案，但理想情况下仍然以 GitOps 方式工作的东西。这意味着最好不要有一个单独的过程来定义 Secret。我会告诉你如何做到这一点。

## GitOps 方式的 Secret

有两种流行的方法可以解决这个问题。它们的工作方式完全不同，但都实现了相同的结果：将 Secret 或其引用存储在 Git 存储库中的能力。你选择哪一种将取决于你的需求。我们来聊聊此二者。

### SealedSecrets

我们已经确定你不能在 Git 存储库中以纯文本形式存储 Secret 。但是如何将它们存储在非纯文本版本中呢？这正是 *SealedSecrets* 工具所做的。它允许你加密你的 Secret，并且只将它们的加密版本存储在你的 Git 存储库中。就那么简单。

你问 *SealedSecrets* 是如何工作的？你在 Kubernetes 集群上安装 *SealedSecrets Controller*，在本地机器上安装 *kubeseal* 二进制文件。*SealedSecrets* 将生成用于加密 Secret 的私钥和公钥。在将密钥提交到 Git 存储库之前，你将使用 *kubeseal* 二进制文件对其进行加密。然后，以加密形式将其存储在存储库中是完全安全的，只有运行在 Kubernetes 集群中的 *SealedSecrets Controller* 才能解密。

#### 如何使用 SealedSecrets

首先，按照此处的 *SealedSecrets* 安装说明进行操作。 启动并运行它后，你可以尝试使用 *kubeseal* 密封你的第一个 Secret。让我们创建一个简单的 Kubernetes  Secret 定义 YAML 文件并使用 *kubeseal* 对其进行密封。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
type: Opaque
data:
  username: my-username
  password: super-secret-password
```

获得文件后，你可以将其内容传给 *kubeseal*：

```bash
cat secret.yaml| kubeseal --controller-name=sealed-secrets-controller --format yaml > sealed-secret.yaml
```

如果你现在查看创建的 seal-secret.yaml 文件，你会看到实际的用户名和密码已加密。

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: example-secret
  namespace: default
spec:
  encryptedData:
    password: AgC7jlVk(...)eb+XOk5/99fKHk=
    username: AgAHbCU7(...)hIgv5D6LDYopF4n
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: example-secret
      namespace: default
    type: Opaque
```

该文件现在可以安全地存储在 Git 存储库中，因为只有用于加密该文件的 *SealedSecrets Controller* 才能解密它。

但是你如何在你的集群中使用这个 Secret 呢？这很简单。你可以直接将该密封文件应用到你的集群，运行在其上的 *SealedSecrets* 控制器将自动解封它并从中创建一个标准的 Kubernetes  Secret 资源。

```bash
$ kubectl apply -f sealed-secret.yaml
sealedsecret.bitnami.com/example-secret created

$ kubectl get secret
NAME                                 TYPE                 DATA   AGE
example-secret                       Opaque               2      7s
```

从现在开始，你可以像使用任何标准 Kubernetes 密钥一样使用 example-secret。

### ExternalSecrets

为你的 GitOps 需求存储 Secret 的另一种方法是使用 *ExternalSecrets*。它的工作方式与 *SealedSecrets* 不同，但也解决了在 Git 存储库中存储纯文本 Secret 的问题。*ExternalSecrets* 通过消除将实际 Secret 存储在存储库中的需要来做到这一点。相反，你的 Secret 可以安全地存储在 Secret 保险库中，你只需要将对 Secret 的引用存储在你的存储库中。

因此，例如，在 Git 中的文件中没有实际的用户名和密码，而是有一个文件，上面写着“这个用户名是密码存储在此 key 下的 Secret 保险库中”。然后，*External Secret Operator* 的工作就是在你需要时为你获取实际值。

#### 使用 ESO（External Secret Operator）

可以像使用 Helm 的任何其他工具一样安装 *External Secret Operator*。你可以按照此处的安装和初始配置步骤进行操作。启动并运行 *ExternalSecrets* 后，使用它就非常简单。你首先需要将你的 Secret 添加到要使用的 Secret 保险库中，然后创建一个 *ExternalSecrets* 引用文件。此文件将替代你的典型 Kubernetes  Secret 定义文件。

如前所述，使用 ESO 意味着从外部 Secret 保险库中引用实际 Secret 。因此，你创建了一个外部 Secret 资源，ESO 将从后台的外部保险库中获取实际 Secret，并为你创建一个实际的 Kubernetes Secret。这是一个例子：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-externalsecret
spec:
  refreshInterval: 3h
  secretStoreRef:
    name: azure-keyvault
    kind: SecretStore
  target:
    name: database-secret
    creationPolicy: Owner
  data:
  - secretKey: database-secret-dev
    remoteRef:
      key: database-secret-dev
```

这是一个 *ExternalSecrets* 定义文件，它告诉 ESO 从 Azure Key Vault 获取 **database-secret-dev** 密钥的值，并从中创建一个名为 **database-secret** 的 Kubernetes 密钥。如你所见，我们在此文件中没有实际的 Secret 值，因此将其存储在 Git 存储库中非常好。

在使用 Secret 方面也是如此。你只需将该 *ExternalSecrets* 定义文件应用到你的集群，ESO 就会自动从定义的 Secret 保险库中获取 Secret 并从中创建一个实际的 Kubernetes  Secret 。
```bash
$ kubectl apply -f external-secret.yaml
externalsecret.external-secrets.io/database-externalsecret created

$ kubectl get secret
NAME                                 TYPE                 DATA   AGE
example-secret                       Opaque               2      12m
database-secret                      Opaque               2      4s
```

## 总结

如你所见，GitOps  Secret 问题可以得到解决。这甚至没有那么困难。但是，它确实包含一些额外的步骤和工具。但是一旦完成初始设置，在 GitOps 实践中安全地管理 Secret 就不需要其他的日常工作。

如果你想了解更多有关 Secret 或 GitOps 的信息，可以在[我们的博客](https://releasehub.com/blog)上找到更多内容。