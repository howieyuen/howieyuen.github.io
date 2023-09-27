---
author: howieyuen
date: 2022-11-02
title: Secrets Store CSI Driver
tag: [SaC, secret, csi]
---

## 概述

Secrets Store CSI Driver 通过[容器存储接口 (CSI)](https://kubernetes-csi.github.io/docs/) 卷将 Secret 存储与 Kubernetes 集成。

Secrets Store CSI Driver `secrets-store.csi.k8s.io` 允许 Kubernetes 将外部机密存储中的多个 Secret、密钥和证书作为卷挂载到其 Pod 中。
附加卷后，其中的数据将被挂载到容器的文件系统中。

Secrets Store CSI Driver 是一个 DaemonSet，可促进与每个 Kubelet 实例的通信。每个驱动程序 Pod 都有以下容器：

- node-driver-registrar：负责向 Kubelet 注册 CSI 驱动程序，以便它知道在哪个 unix 域套接字上发出 CSI 调用。
  此 sidecar 容器由 Kubernetes CSI 团队提供。
- secrets-store：实现 CSI 规范中描述的 CSI 节点服务 gRPC 服务。
  它负责在 Pod 创建/删除期间挂载/卸载卷。
  此组件在[secrets-store-csi-driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) 仓库中开发和维护。
- liveness-probe：负责监控 CSI 驱动的健康状况并向 Kubernetes 报告。
  这使 Kubernetes 能够自动检测驱动程序的问题并重新启动 Pod 以尝试修复问题。
  此 sidecar 容器由 Kubernetes CSI 团队提供。

## Provider

CSI Driver 使用 gRPC 与 Provider 通信，以从外部 Secrets Store 获取挂载内容。
有关如何为 Driver 实现 Provider 和支的持 Provider 的标准的更多详细信息，请参阅[Providers](https://secrets-store-csi-driver.sigs.k8s.io/providers.html)。

当前支持的 Provider：
- [AWS Provider](https://github.com/aws/secrets-store-csi-driver-provider-aws)
- [Azure Provider](https://azure.github.io/secrets-store-csi-driver-provider-azure/)
- [GCP Provider](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp)
- [Vault Provider](https://github.com/hashicorp/secrets-store-csi-driver-provider-vault)

## 原理

与 Kubernetes Secret 类似，在 Pod 启动和重新启动时，Secrets Store CSI Driver 使用 gRPC 与 Provider 通信，
从 `SecretProviderClass` 自定义资源中指定的外部 Secrets Store 检索 Secret 内容。然后将卷作为 tmpfs 挂载到 Pod 中，
并将 Secret 内容写入该卷。在 Pod delete 时，会清理并删除相应的卷。

![](/secret-as-code/secrets-store-csi-driver.png)

## 演示（Vault）

### 安装 secrets-store-csi-driver 
{{< details title="install csi-secrets-store" open=true >}}
```bash
helm repo add secrets-store-csi-driver \
https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm install csi-secrets-store \
secrets-store-csi-driver/secrets-store-csi-driver \
--namespace kube-system
```
{{< /details >}}

### 安装 Vault
{{< details title="install vault" open=true >}}
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault \
    --set "server.dev.enabled=true" \
    --set "injector.enabled=false" \
    --set "csi.enabled=true"
```
{{< /details >}}

### 配置 Vault
{{< details title="config vault" open=true >}}
```bash
kubectl exec -it vault-0 -- /bin/sh

# 在 Vault 中配置敏感信息
vault kv put secret/db-pass password="db-secret-password"

# 启用 Kubernetes 身份验证
vault auth enable kubernetes

# 配置 kubernetes 身份认证规则
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# 设置读权限的 policy
vault policy write internal-app - <<EOF
path "secret/data/db-pass" {
  capabilities = ["read"]
}
EOF

# 创建 role ，关联上一步创建的 policy
vault write auth/kubernetes/role/database \
    bound_service_account_names=webapp-sa \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=20m
```
{{< /details >}}

### 创建 SecretProviderClass
{{< details title="secret-provider-class-vault.yaml" open=true >}}
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-database
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.default:8200"
    roleName: "database"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/db-pass"
        secretKey: "password"
```
{{< /details >}}

### 创建 Pod
{{< details title="pod.yaml" open=true >}}
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: webapp
spec:
  serviceAccountName: webapp-sa
  containers:
  - image: jweissig/app:0.0.1
    name: webapp
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "vault-database"
```
{{< /details>}}

## 参考资料
- https://github.com/kubernetes-sigs/secrets-store-csi-driver
- https://secrets-store-csi-driver.sigs.k8s.io/concepts.html
- https://secrets-store-csi-driver.sigs.k8s.io/providers.html
- https://learn.hashicorp.com/tutorials/vault/kubernetes-secret-store-driver