---
author: howieyuen
date: 2023-09-27
title: Helm Secrets
tag: [SaC, helm, secret]
---

## 概述

Helm Secrets 是 Helm 的一个插件，能够利用 Helm 模板化 Secrets 资源。它使用 [SOPS（Mozilla 研发）](https://github.com/mozilla/sops)来加密 Secret。SOPS 是一个加密文件的编辑器，采用的是非对称加密，支持 YAML、JSON、ENV、INI 和二进制格式，并支持 AWS KMS、GCP KMS、Azure Key Vault 和 PGP 进行加密。

Helm Secrets 还支持其他后端，例如：vals，它是一种用于管理来自各种来源的配置值和秘密的工具。目前已经支持：
- [Vault](https://github.com/variantdev/vals#vault)
- [AWS SSM Parameter Store](https://github.com/variantdev/vals#aws-ssm-parameter-store)
- [AWS Secrets Manager](https://github.com/variantdev/vals#aws-secrets-manager)
- [AWS S3](https://github.com/variantdev/vals#aws-s3)
- [GCP Secrets Manager](https://github.com/variantdev/vals#gcp-secrets-manager)
- [Azure Key Vault](https://github.com/variantdev/vals#azure-key-vault)
- [SOPS-encrypted files](https://github.com/variantdev/vals#sops)
- [Terraform State](https://github.com/variantdev/vals#terraform-tfstate)
- [Plain File](https://github.com/variantdev/vals#file)

下文以 PGP 方式为例，进行说明。

## 安装

gpg:
```bash
brew install gpg
```

sops:
```bash
brew install sops
```

helm-secrets:
```bash
helm plugin install https://github.com/jkroepke/helm-secrets --version v3.12.0
```

## 原理

![](/secret-as-code/helm-secrets.png)

### 加密

1. 加密敏感信息文件 secret.yaml
2. 调用 helm secrets enc命令
3. 读取 gpg 公钥
4. 加密结果覆盖源文件

### 解密

5. 解密文件 secret.yaml
6. 调用 helm secrets dec命令
7. 读取 gpg 私钥
8. 解密结果写入新文件 secret.yaml.dec


## 参考资料

- https://github.com/jkroepke/helm-secrets
- https://github.com/jkroepke/helm-secrets/blob/main/docs/Usage.md
- https://github.com/mozilla/sops
- https://github.com/variantdev/vals
