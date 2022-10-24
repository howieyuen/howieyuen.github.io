---
author: howieyuen
date: 2022-10-15
title: Sealed Secrets
tag: [iac, secret]
---

# 概述

Sealed Secrets 由两个部分组成：
- 集群侧控制器：`sealed-secrets-controller`
- 用户侧工具：`kubeseal`

kubeseal 使用非对称加密算法，加密 Secret，加密结果仅有 sealed-secrets-controller 才能解密。
加密后的 `Secret` 编码在 SealedSecret 资源中，详细结构如下：

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: mynamespace
spec:
  encryptedData:
    foo: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq.....
```

解密后的 `Secret` 如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mynamespace
data:
  foo: YmFy  # <- base64 encoded "bar"
```

`SealedSecret` 和 `Secret` 的关系类似于 `Deployment` 和 `Pod`，`SealedSecret` 有个 `template` 字段，是生成的 `Secret` 的模板；
此二者之间的 `labels` 和 `annotations` 并不要求完成一致。
最终生成的 `Secret` 与 `SealedSecret` 相对独立，但 `SealedSecret` 的更新或删除，会连带到生成的 `Secret`。

# 安装

## kubeseal

```bash
brew install kubeseal
```

## sealed-secrets-controller

kubeseal 默认尝试连接名为 `sealed-secrets-controller` 的控制器；在使用时，可以通过 `--controller-name` 传递名称，也可以在安装控制器时，通过 `--fullnameOverride` 指定控制器名称：

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets -n kube-system --set-string fullnameOverride=sealed-secrets-controller sealed-secrets/sealed-secrets 
```

# 原理

![](/secret-as-code/sealedsecrets.png)

## 加密

Step 1-5：使用 kubeseal 创建 SealedSecret 到本地

```bash
# 创建 JSON 格式的 k8s Secret
echo -n bar | kubectl create secret generic mysecret --dry-run=client --from-file=foo=/dev/stdin -o json >mysecret.json
# kubeseal 加密 k8s Secret
kubeseal < mysecret.json > mysealedsecret.json
```

1. kubeseal 加密 k8s Secret
2. kubeseal 请求 sealed-secrets-controller 加密
3. sealed-secrets-controller 使用公钥加密
4. 加密结果返回，写入本地文件 mysealedsecret.json

{{< hint info >}}
公钥/私钥保存在同 Namespace 下的一个 Secret 中，可通过指定 label 为 `sealedsecrets.bitnami.com/sealed-secrets-key` 查询
{{< /hint >}}

## 解密
Step 5-8：使用 `kubectl` 创建 `SealedSecret` 到集群

```bash
kubectl create -f mysealedsecret.json
```

5. 下发 mysealedsecret.json
6. `sealed-secrets-controller` 监听 CRD，发现新对象
7. `sealed-secrets-controller` 使用私钥解密
8. 生成 k8s `Secret`，并加入集群

# 总结

每个 SealedSecret 都使用其自己的随机非对称密钥加密，该密钥特定于 SealedSecret 名称和 Namespace。将加密数据复制粘贴到另一个 Secret 或另一个 Namespace 中将不起作用。

- ✅ 设置和使用简单
- ✅ 正确成熟且维护良好的解决方案
- ❎ 常规 Secret 仍然暴露
- ❎ 不能在 Secret 之外使用（例如 ConfigMap）

# 参考资料

- https://github.com/bitnami-labs/sealed-secrets
- https://aws.amazon.com/cn/blogs/china/managing-secrets-deployment-in-kubernetes-using-sealed-secrets/
- https://learnk8s.io/kubernetes-secrets-in-git
