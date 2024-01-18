---
author: howieyuen
date: 2022-11-2
title: External Secrets Operator
tag: [sac, secret, operator]
---

## 概述

External Secrets Operator（ESO）是一个 Kubernetes Operator，它支持：

- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [HashiCorp Vault](https://www.vaultproject.io/)
- [Google Secrets Manager](https://cloud.google.com/secret-manager)
- [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/)
- [IBM Cloud Secret Manager](https://www.ibm.com/cloud/secrets-manager)
- ...

等外部 Secret 管理系统。Operator 从外部 API 读取信息并自动将值注入 Kubernetes Secret。
ESO 的目标是将来自外部 API 的机密信息同步到 Kubernetes。ESO 是自定义 API 资源的集合：

- SecretStore
- ExternalSecret
- ClusterSecretStore
- ClusterExternalSecret

它们为外部 API 提供了一个用户友好的抽象，帮助用户存储 Secret 并管理它的生命周期。

### SecretStore

`SecretStore` 将身份验证/访问的关注点与工作负载所需的实际 `Secret` 和配置分开。`ExternalSecret` 指定要获取的内容，`SecretStore` 指定如何访问。Namespace 级别。

下面的代码示例是以 Vault 作为后端，使用静态 token：

### ExternalSecret

`ExternalSecret` 声明要获取的数据。它引用了一个知道如何访问该数据的 `SecretStore`。控制器使用该 `ExternalSecret` 作为创建 `Secret` 的蓝本。Namespace 级别。

### ClusterSecretStore

`ClusterSecretStore` 是一个全局的、集群范围的 `SecretStore`，可以从所有命名空间中引用。可以将它作为 Secret Provider 的中央网关。

### ClusterExternalSecret

`ClusterExternalSecret` 是一个集群范围的资源，可用于将 `ExternalSecret` 推送到特定的命名空间。
使用 `namespaceSelector` 可以选择命名空间，并且任何匹配的命名空间都将具有在其中创建的 `externalSecretSpec` 中指定的 `ExternalSecret`。

## 原理

ESO 按照以下列方式同步 ExternalSecrets：

1. ESO 使用 `spec.secretStoreRef` 来查找合适的 `SecretStore`。如果不存在或 `spec.controller` 字段不匹配，将丢弃不再处理。
2. ESO 使用 `SecretStore.spec` 的指定凭据实例化外部 API 客户端。
3. ESO 根据 `ExternalSecret` 的要求获取机密值，如果有必要，将对机密信息解码。
4. ESO 根据 `ExternalSecret.target.template` 提供的模板创建 `Kind=Secret`。可以使用来自外部 API 的机密值对 `Secret.data` 进行模板化。
5. ESO 确保机密值与外部 API 保持同步。

![](/secret-as-code/eso.png)

## 安装

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace \
    --set installCRDs=true
```

## 演示

1、安装 vault
``` bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

2、准备数据
```bash
kubectl exec -it vault-0 -- /bin/sh
vault kv put secret/foo my-value=s3cr3t
```

3、创建 SecretStore，引用 Vault
{{< details title="secretstore-vault.yaml" open=true >}}
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      # vault kv put secret/foo my-value=s3cr3t
      server: "http://vault.default.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        # points to a secret that contains a vault token
        # https://www.vaultproject.io/docs/auth/token
        tokenSecretRef:
          name: "vault-token"
          key: "token"
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
data:
  token: cm9vdA== # "root"
```
{{< /details >}}

4、创建 ExternalSecret
{{< details title="externalsecret-vault.yaml" open=true >}}
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: example-of-vault-sync
  data:
  - secretKey: foobar
    remoteRef:
      key: secret/foo
      property: my-value
```
{{< /details >}}

5、验证同步结果
```bash
$ kubectl get secret example-of-vault-sync -ojsonpath='{.data.foobar}'|base64 -d
s3cr3t
```

## 参考资料

- https://github.com/external-secrets/external-secrets
- https://external-secrets.io/v0.6.1/
- https://www.infoq.com/articles/k8s-external-secrets-operator/
