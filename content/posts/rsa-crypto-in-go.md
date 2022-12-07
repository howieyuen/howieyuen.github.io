---
title: Golang 中的 RSA 加解密算法
date: 2022-11-01
tags: [go, rsa, crypto]
categories: [crypto]
---

本文受到 Gitlab 中 [`crypgo`](https://gitlab.com/rahasak-labs/crypgo) 项目启发，
作者在 Medium 中发表了一篇博文，[RSA cryptography in Golang](https://medium.com/rahasak/golang-rsa-cryptography-1f1897ada311) 作为说明。
读完后发现代码中存在一些小问题，因此重整代码实现，使其更加实用。

<!--more-->

> 仓库地址：[https://github.com/howieyuen/crypto](https://github.com/howieyuen/crypto)

## RSA 加解密简介

RSA 加密算法是目前最有影响力的公钥加密算法，并且被普遍认为是目前最优秀的公钥方案之一。
RSA 是第一个能同时用于加密和数宇签名的算法，它能够抵抗到目前为止已知的所有密码攻击，已被 ISO 推荐为公钥数据加密标准。
RSA 加密算法基于一个十分简单的数论事实：将两个大素数相乘十分容易，但想要对其乘积进行因式分解却极其困难，因此可以将乘积公开作为加密密钥。

## RSA加密、签名区别

加密和签名都是为了安全性考虑，但略有不同。常有人问加密和签名是用私钥还是公钥？其实都是对加密和签名的作用有所混淆。
简单的说，加密是为了防止信息被泄露，而签名是为了防止信息被篡改。这里举 2 个例子说明。

**场景 1：战场上，B要给A传递一条消息，内容为某一指令。**

RSA 的加密/解密过程如下：
- A 生成一对密钥（公钥和私钥），私钥不公开，A 自己保留。公钥为公开的，任何人可以获取。
- A 传递自己的公钥给 B，B 用 A 的公钥对消息进行加密。
- A 接收到 B 加密的消息，利用 A 自己的私钥对消息进行解密。

在这个过程中，只有 2 次传递过程，第一次是 A 传递公钥给 B，第二次是 B 传递加密消息给 A。
即使都被敌方截获，也没有危险性，因为只有A的私钥才能对消息进行解密，防止了消息内容的泄露。

**场景 2：A 收到 B 发的消息后，需要回复“收到”。**

RSA 签名/验签的过程如下：
- A 生成一对密钥（公钥和私钥），私钥不公开，A 自己保留。公钥为公开的，任何人可以获取。
- A 用自己的私钥对消息加签，形成签名，并将加签的消息和消息本身一起传递给 B。
- B 收到消息后，在获取 A 的公钥进行验签，如果验签出来的内容与消息本身一致，证明消息是 A 回复的。

在这个过程中，只有2次传递过程，第一次是 A 传递加签的消息和消息本身给 B，第二次是 B 获取 A 的公钥。
即使都被敌方截获，也没有危险性，因为只有 A 的私钥才能对消息进行签名，即使知道了消息内容，也无法伪造带签名的回复给 B，防止了消息内容的篡改。

但是，综合两个场景你会发现，第一个场景虽然被截获的消息没有泄露，但是可以利用截获的公钥，将假指令进行加密，然后传递给 A。
第二个场景虽然截获的消息不能被篡改，但是消息的内容可以利用公钥验签来获得，并不能防止泄露。
所以在实际应用中，要根据情况使用，也可以同时使用加密和签名。
比如 A 和 B 都有一套自己的公钥和私钥，当 A 要给 B 发送消息时，先用 B 的公钥对消息加密，再对加密的消息使用 A 的私钥加签名，达到既不泄露也不被篡改，更能保证消息的安全性。

总结：公钥加密、私钥解密、私钥签名、公钥验签。

## 代码实现

首先定义 `Config` 结构体，用来声明公私钥的目录名、文件名和长度：

```go
type Config struct {
    DotKeys  string // DotKeys represents the parent dir of IDRsa and IDRsaPub
    IDRsa    string // IDRsa represents private key file
    IDRsaPub string // IDRsaPub represents public key file
    KeySize  int    // KeySize represents the bit size of key pair
}
```

生成密钥对：通过 `Config` 生成
```go
func (c *Config) GenerateKeyPair() (*rsa.PrivateKey, error) 
```

保存密钥对：按照声明的文件名保存到指定目录
```go
func (c *Config) SaveKeyPair(keyPair *rsa.PrivateKey) error
```

再定义 `KeyPair` 结构体用来保存公私钥文件内容：
```go
type KeyPair struct {
	publicKey  *rsa.PublicKey
	privateKey *rsa.PrivateKey
}
```

读取公钥：
```go
func (c *Config) LoadPublicKey() (*KeyPair, error)
```

公钥加密：有两种算法可选，OAEP 和 PKCS：
```go
func (p *KeyPair) EncryptPKCS1v15(plainText string) (string, error)

func (p *KeyPair) EncryptOAEP(plainText string) (string, error)
```

读取私钥：
```go
func (c *Config) LoadPrivateKey() (*KeyPair, error)
```

私钥解密：有两种算法可选，OAEP 和 PKCS，解密与加密使用算法保持一致
```go
func (p *KeyPair) DecryptPKCS1v15(cipherText string) (string, error)

func (p *KeyPair) DecryptOAEP(cipherText string) (string, error)
```

私钥签名：有两种方式可选，PKCS 和 PSS
```go
func (p *KeyPair) SignPKCS1v15(payload string) (string, error)

func (p *KeyPair) SignPSS(payload string) (string, error)
```

公钥验签：有两种方式可选，PKCS 和 PSS，验签和签名的算法保持一致
```go
func (p *KeyPair) VerifyPKCS1v15(payload, signature64 string) error 

func (p *KeyPair) VerifyPSS(payload, signature64 string) error
```

## 明文长度

仓库中默认生成的密钥长度是 1024 位，RSA 算法本身要求加密内容也就是明文长度（m）必须小于密钥长度（n），即 0 < m < n。
如果小于这个长度就需要进行填充（Padding），因为如果没有 Padding，就无法确定解密后内容的真实长度。
字符串之类的内容问题还不大，以 0 作为结束符，但对二进制数据就很难，因为不确定后面的 0 是内容还是内容结束符。
而只要用到 Padding，那么就要占用实际的明文长度，于是实际明文长度需要减去 Padding 字节长度。
我们一般使用的 Padding 标准有 NoPadding、OAEPPadding、PKCS1Padding 等，其中 PKCS#1 建议的 Padding 就占用了 11 个字节。

这样，对于 1024 长度的密钥。128 字节（1024 bit）减去 11 字节正好是 117 字节。
但对于 RSA 加密来讲，Padding 也是参与加密的，所以依然按照 1024 bit 去理解，但实际的明文只有 117 字节。

因此对于长度大于 117 字节的明文，需要分段加密，然后拼接：

```go
func split(buf []byte, limit int) [][]byte {
	var chunk []byte
	chunks := make([][]byte, 0, len(buf)/limit+1)
	for len(buf) >= limit {
		chunk, buf = buf[:limit], buf[limit:]
		chunks = append(chunks, chunk)
	}
	if len(buf) > 0 {
		chunks = append(chunks, buf[:])
	}
	return chunks
```

## 参考资料

- [RSA加密、解密、签名、验签的原理及方法](https://www.cnblogs.com/pcheng/p/9629621.html)
- [RSA密钥长度、明文长度和密文长度](https://cloud.tencent.com/developer/article/1199963)
- [RSA加密解密（无数据大小限制，php、go、java互通实现）](https://segmentfault.com/a/1190000011263680)