---
author: howieyuen
date: 2023-09-27
title: Kamus
tag: [KamusSecret]
categories: [SaC]
---

## 概述

Kamus 架构似于 Sealed Secrets 和 Helm Secrets。但是，Kamus 使你可以加密特定应用程序的 Secret，并且只有该应用程序可以解密。细化权限使 Kamus 更适合具有高安全标准的零信任环境。Kamus 通过关联 ServiceAccount 和你的 Secret 工作，仅有使用此服务帐户运行的应用程序对其进行解密。

Kamus 由三个模块组成：
- 加密 API
- 解密 API
- 密钥管理系统（KMS）

加密和解密 API 处理加密和解密请求。KMS 是各种加密解决方案的包装器。目前支持：
- AES：对所有 Secret 使用一个密钥
- AWS KMS、Azure KeyVault、Google Cloud KMS：为每个服务帐户创建一个密钥。

Kamus 附带 3 个实用程序，使其更易于使用：
- CLI：一个小型 CLI，可简化与 Encrypt API 的交互。
- init 容器：一个与 Decrypt API 交互的初始化容器。
- CRD 控制器：允许使用 Kamus 创建本集群的 k8s Secret。

## 原理

![](/secret-as-code/kamus.png)

### 加密

1. 用户加密敏感信息（字符串，不是对象）
2. kamus-cli 请求 kamus-encrytptor 加密
3. kamus-controller 提供公钥
4. 加密结果返回

### 解密

5. 用户下发 KamusSecret 类型的 CR，包含加密字段（kamus-secret.yaml）
6. kamus-decryptor 发现新的 KamusSecret 字段，请求 kamus-controller 解密
7. kamus-controller 提供私钥
8. 解密成功，生成 k8s secret

## 安装

### controller/decryptor/encryptor

```bash
helm repo add soluto https://charts.soluto.io
helm upgrade --install kamus soluto/kamus
```

### kamus-cli

```bash
npm install -g @soluto-asurion/kamus-cli
```

## 演示

### KamusSecret

```bash
# 端口映射
$ kubectl port-forward svc/kamus-encryptor 9999:80

# 创建 ServiceAccount
$ kubectl create sa howieyuen

# 加密 zhangsan
$ kamus-cli encrypt -s zhangsan -a howieyuen -n default -u http://localhost:9999 --allow-insecure-url
[info  kamus-cli]: Encryption started...
[info  kamus-cli]: service account: howieyuen
[info  kamus-cli]: namespace: default
[warn  kamus-cli]: Auth options were not provided, will try to encrypt without authentication to kamus
[info  kamus-cli]: Successfully encrypted data to howieyuen service account in default namespace
[info  kamus-cli]: Encrypted data:
/QjBaRgthEpEG9z6xO2p2Q==:R+wtZ4ljx3ZR6Wzqrz7DGg==

# 加密 qwer1234 
$ kamus-cli encrypt -s qwer1234 -a howieyuen -n default --allow-insecure-url -u http://localhost:9999
[info  kamus-cli]: Encryption started...
[info  kamus-cli]: service account: howieyuen
[info  kamus-cli]: namespace: default
[warn  kamus-cli]: Auth options were not provided, will try to encrypt without authentication to kamus
[info  kamus-cli]: Successfully encrypted data to howieyuen service account in default namespace
[info  kamus-cli]: Encrypted data:
YGSgyn2hMbmcuwvc58JJSQ==:LCglu6OQ2pMXLiHGUpNdZg==

# 创建 kamus-secret.yaml
$ cat > kamus-secret.yaml << EOF
apiVersion: "soluto.com/v1alpha2"
kind: KamusSecret
metadata:
  name: kamus-secret
type: Generic
stringData:
  username: /QjBaRgthEpEG9z6xO2p2Q==:R+wtZ4ljx3ZR6Wzqrz7DGg==
  password: YGSgyn2hMbmcuwvc58JJSQ==:LCglu6OQ2pMXLiHGUpNdZg==
serviceAccount: howieyuen
EOF

# 创建 KamusSecret
$ kubectl apply -f kamus-secret.yaml

# 检查解密结果
$ kubectl get secret kamus-secret -ojsonpath='{.data.username}' |base64 -d
zhangsan
$ kubectl get secret kamus-secret -ojsonpath='{.data.paasowrd}' |base64 -d
qwer1234
```

### 零信任模式

```bash
# 创建包含加密数据的 ConfigMap
$ cat > kamus-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kamus-configmap
  namespace: default
data:
  username: /QjBaRgthEpEG9z6xO2p2Q==:R+wtZ4ljx3ZR6Wzqrz7DGg==
  password: YGSgyn2hMbmcuwvc58JJSQ==:LCglu6OQ2pMXLiHGUpNdZg==
EOF
$ kubectl apply -f kamus-configmap.yaml

# 创建引用 ConfigMap 的 Pod
$ cat > kamus-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: kamus-zero-trust-example
spec:
   serviceAccountName: howieyuen
   automountServiceAccountToken: true
   initContainers:
   - name: "kamus-init"
     image: "soluto/kamus-init-container:latest"
     imagePullPolicy: IfNotPresent
     env:
     - name: KAMUS_URL
       value: http://kamus-decryptor.default.svc.cluster.local/ 
     volumeMounts:
     - name: encrypted-secrets
       mountPath: /encrypted-secrets
     - name: decrypted-secrets
       mountPath: /decrypted-secrets
     args: ["-e","/encrypted-secrets","-d","/decrypted-secrets", "-n", "config.json"]
   containers:
   - name: app
     image: busybox
     imagePullPolicy: IfNotPresent
     args: ["sleep", "3600"]
     volumeMounts:
     - name: decrypted-secrets
       mountPath: /secrets
   volumes:
   - name: encrypted-secrets
     configMap: 
       name: kamus-configmap
   - name: decrypted-secrets
     emptyDir:
       medium: Memory
EOF
$ kubectl apply -f kamus-pod.yaml

# 检查挂载结果
$ kubectl exec kamus-zero-trust-example -c app -- more /secrets/config.json 
{
    "password":"qwer1234",
    "username":"zhangsan"
}
```

## 总结

对于 KamusSecret（第一种方法），kamus-cli 命令行的 --service-account 和 --namespace 参数虽然是强制性的，但纯粹是任意的：使用 KamusSecret 的加密数据不链接到特定 ServiceAccount 和/或 Namespace，并且它们的存在不是加密时的要求。

相反，在零信任模式（第二种方法）的情况下，使 ServiceAccount 和 Namespace 在加密步骤和 initContainer 内部使用的步骤之间保持一致是必须的。使用另一个 ServiceAccount 或另一个 Namespace 不会触发正确的解密。

优势：
- 多种操作模式
- 在零信任环境中工作
- 可以使用真正的 KMS 后端
  
劣势：
- 文档不全
- 处于早期阶段
- 基于 npm 

## 参考资料
- https://github.com/Soluto/kamus
- https://learnk8s.io/kubernetes-secrets-in-git
- https://en.sokube.ch/post/lightweight-kubernetes-gitops-secrets-1