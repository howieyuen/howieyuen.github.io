---
author: Yuan Hao
date: 2020-11-22
title: 认证机制
tag: [kube-apiserver, authenticate]
---

# 1. Kubernetes 中的用户

所有 Kubernetes 集群都有两类用户：由 Kubernetes 管理的 ServiceAccount 和普通用户。

对于与普通用户，Kuernetes 使用以下方式管理：

- 负责分发私钥的管理员
- 类似 Keystone 或者 Google Accounts 这类用户数据库
- 包含用户名和密码列表的文件

因此，kubernetes 并不提供普通用户的定义，普通用户是无法通过 API 调用写入到集群中的。

尽管如此，通过集群的证书机构签名的合法证书的用户，kubernetes 依旧可以认为是合法用户。基于此，kubernetes 使用证书中的 `subject.CommonName` 字段来确定用户名，接下来，通过 RBAC 确认用户对某资源是否存在要求的操作权限。

与此不同的 ServiceAccount，与 Namespace 绑定，与一组 Secret 所包含的凭据有关。这些凭据会挂载到 Pod 中，从而允许访问 kubernetes 的 API。

API 请求要么与普通用户相关，要么与 ServiceAccount 相关，其他的视为匿名请求。这意味着集群内和集群外的每个进程向 kube-apiserver 发起请求时，都必须通过身份认证，否则会被视为匿名用户。

# 2. 认证机制

目前 kubernetes 提供的认证机制丰富多样，尤其是身份验证，更是五花八门：

- 身份验证
  - X509 Client Cert
  - Static Token File
  - Bootstrap Tokens
  - ~~Static Password File~~（deprecated in v1.16）
  - ServiceAccount Token
  - OpenID Connect Token
  - Webhook Token
  - Authentication Proxy
- 匿名请求
- 用户伪装
- client-go 凭据插件

## 2.1 身份验证策略





### 2.1.1 X509 Client Cert

X509 客户端证书认证，也被称为 TLS 双向认证，即为服务端和客户端互相验证证书的正确性。使用此认证方式，只要是 CA 签名过的证书都能通过认证。

1. 启用

   kube-apiserver 通过指定 `--client-ca-file` 参数启用此认证方式。

2. 认证接口

   ```go
   // staging/src/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go
   type Request interface {
   	AuthenticateRequest(req *http.Request) (*Response, bool, error)
   }
   ```

   该方法接收客户端请求。若验证失败，bool 返回 false，验证成功，bool 返回 true，Response 中携带身份验证用户的信息，例如 Name、UID、Groups、Extra。

3. 认证实现

   ```go
   // staging/src/k8s.io/apiserver/pkg/authentication/request/x509/x509.go
   func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
   	if req.TLS == nil || len(req.TLS.PeerCertificates) == 0 {
   		return nil, false, nil
   	}
   
   	// Use intermediates, if provided
   	optsCopy, ok := a.verifyOptionsFn()
   	// if there are intentionally no verify options, then we cannot authenticate this request
   	if !ok {
   		return nil, false, nil
   	}
   	if optsCopy.Intermediates == nil && len(req.TLS.PeerCertificates) > 1 {
   		optsCopy.Intermediates = x509.NewCertPool()
   		for _, intermediate := range req.TLS.PeerCertificates[1:] {
   			optsCopy.Intermediates.AddCert(intermediate)
   		}
   	}
   
   	remaining := req.TLS.PeerCertificates[0].NotAfter.Sub(time.Now())
   	clientCertificateExpirationHistogram.Observe(remaining.Seconds())
       // 校验证书，如果通过，可解析 user 信息
   	chains, err := req.TLS.PeerCertificates[0].Verify(optsCopy)
   	if err != nil {
   		return nil, false, err
   	}
   
   	var errlist []error
   	for _, chain := range chains {
   		user, ok, err := a.user.User(chain)
   		if err != nil {
   			errlist = append(errlist, err)
   			continue
   		}
   
   		if ok {
   			return user, ok, err
   		}
   	}
   	return nil, false, utilerrors.NewAggregate(errlist)
   }
   ```

   

### 2.1.2 Static Token File

Token 也被称为令牌，服务端为了验证客户端身份，需要客户端向服务端提供一个可靠的验证信息，这个验证信息就是 Token。目前，令牌会长期有效，并且在不重启 API 服务器的情况下 无法更改令牌列表。

1. 启用

   kube-apiserver 通过指定 `--token-auth-file` 参数启用，令牌文件是一个 CSV 文件，包含至少 3 个列：令牌、用户名和用户的 UID。 其余列被视为可选的组名。示例如下：

   ```
   token,user,uid,"group1,group2,group3"
   ```

2. 请求头配置

   在 HTTP 请求头中，设置 Authentication 的值，格式为 Bearer $TOKEN，格式如下：

   ```
   Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
   ```

3. 认证实现

   ```go
   // staging/src/k8s.io/apiserver/pkg/authentication/token/tokenfile/tokenfile.go
   func (a *TokenAuthenticator) AuthenticateToken(ctx context.Context, value string) (*authenticator.Response, bool, error) {
   	user, ok := a.tokens[value]
   	if !ok {
   		return nil, false, nil
   	}
   	return &authenticator.Response{User: user}, true, nil
   }
   ```

   该认证方式相对简单，a.tokens 保存了服务端 Token 列表，通过map查询客户端提供的 Token 是否存在，存在即认证成功，反之则认证失败。

### 2.1.3 Bootstrap Tokens

Bootstrap Token 是一种简单的 Bearer Token，这种令牌是在新建集群或者在现在集群中加入节点时使用。一般是由 kubeadm 管理，以 secret 形式保存在 kube-system 命名空间，可以动态地创建删除，并且 kube-controller-manager 中 TokenCleaner 会在 Token 过期时删除。该能力目前依旧是 alpha 阶段，但官方预期也不会有大的突破性变化。

1. 启用

   kube-apiserver 设置 `--enable-bootstrap-token` 启动 Bootstrap Token 身份认证，并且依赖 kube-controller-manager 设置 `--controllers=*,tokencleaner,bootstrapsigner` 启动 TokenCleaner 和 BootstrapSigner。

2. 请求头配置

   Token 的格式为 `[a-z0-9]{6}.[a-z0-9]{16}`，第一部分是 token id，第二部分是 token 的 secret。可以用如下方式设置 HTTP Header：

   ```
   Authorization: Bearer 781292.db7bc3a58fc5f07e
   ```

3. 认证实现

   ```go
   // plugin/pkg/auth/authenticator/token/bootstrap/bootstrap.go
   func (t *TokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
   	// 1.校验token格式
   	tokenID, tokenSecret, err := bootstraptokenutil.ParseToken(token)
   	if err != nil {
   		return nil, false, nil
   	}
   
   	// 2.拼接secret name，获取secret对象
   	secretName := bootstrapapi.BootstrapTokenSecretPrefix + tokenID
   	secret, err := t.lister.Get(secretName)
   	if err != nil {
   		if errors.IsNotFound(err) {
   			klog.V(3).Infof("No secret of name %s to match bootstrap bearer token", secretName)
   			return nil, false, nil
   		}
   		return nil, false, err
   	}
   
   	// 3.校验secret有效，不在删除中
   	if secret.DeletionTimestamp != nil {
   		tokenErrorf(secret, "is deleted and awaiting removal")
   		return nil, false, nil
   	}
   
   	// 4.校验secret类型必须是 bootstrap.kubernetes.io/token
   	if string(secret.Type) != string(bootstrapapi.SecretTypeBootstrapToken) || secret.Data == nil {
   		tokenErrorf(secret, "has invalid type, expected %s.", bootstrapapi.SecretTypeBootstrapToken)
   		return nil, false, nil
   	}
   
   	// 5.校验token secret有效
   	ts := bootstrapsecretutil.GetData(secret, bootstrapapi.BootstrapTokenSecretKey)
   	if subtle.ConstantTimeCompare([]byte(ts), []byte(tokenSecret)) != 1 {
   		tokenErrorf(secret, "has invalid value for key %s, expected %s.", bootstrapapi.BootstrapTokenSecretKey, tokenSecret)
   		return nil, false, nil
   	}
   
   	// 6.校验token id有效
   	id := bootstrapsecretutil.GetData(secret, bootstrapapi.BootstrapTokenIDKey)
   	if id != tokenID {
   		tokenErrorf(secret, "has invalid value for key %s, expected %s.", bootstrapapi.BootstrapTokenIDKey, tokenID)
   		return nil, false, nil
   	}
   
   	// 7.校验token是否过期
   	if bootstrapsecretutil.HasExpired(secret, time.Now()) {
   		// logging done in isSecretExpired method.
   		return nil, false, nil
   	}
   
   	// 8.校验secret对象的data字段中，key为usage-bootstrap-authentication，value为true
   	if bootstrapsecretutil.GetData(secret, bootstrapapi.BootstrapTokenUsageAuthentication) != "true" {
   		tokenErrorf(secret, "not marked %s=true.", bootstrapapi.BootstrapTokenUsageAuthentication)
   		return nil, false, nil
   	}
   
   	// 9.获取secret.data[auth-extra-groups]，与default group组合
   	groups, err := bootstrapsecretutil.GetGroups(secret)
   	if err != nil {
   		tokenErrorf(secret, "has invalid value for key %s: %v.", bootstrapapi.BootstrapTokenExtraGroupsKey, err)
   		return nil, false, nil
   	}
   
   	return &authenticator.Response{
   		User: &user.DefaultInfo{
   			Name:   bootstrapapi.BootstrapUserPrefix + string(id),
   			Groups: groups,
   		},
   	}, true, nil
   }
   ```

### 2.1.4  ServiceAccount Token

其他认证方式都是从 kubernetes 集群外部访问 kube-apiserver 组件，而 ServiceAccount 是从 Pod 内部访问，提供给 Pod 中的进程使用。ServiceAccount 包含了 3 个部分的内容：

- Namespace：指定 Pod 所在的命名空间
- CA：kube-apiserver CA 公钥证书，是 Pod 内部进程对 kube-apiserver 进行验证的证书
- Token：用于身份验证，通过 kube-apiserver 私钥签发经过 Base64 编码的 Bearer Token

他们都通过 mount 命令挂载到 Pod 的文件系统中，Namespace 存储在 /var/run/secrets/kubernetes.io/serviceaccount/namespace，经过 Base64 加密；CA 的存储路径 /var/run/secrets/kubernetes.io/serviceaccount/ca.crt；Token 存储在 /var/run/secrets/kubernetes.io/serviceaccount/token 文件中。

1. 启用

   kube-apiserver 指定以下参数启用

   - `--service-account-key-file` ：包含用来给 Bearer Token 签名的 PEM 编码密钥，如果未指定，使用 kube-apiserver 的 TLS 私钥。
   - `--service-account-lookup`：用于验证 service account token 是否存在 etcd 中，默认为 true。

2. 配置

   ServiceAccount 通常是 kube-apiserver 自动创建，并通过准入控制器关联到 Pod 中。当然也可以在 Pod.spec.serviceAccountName 显示地指定。

3. 认证实现

   ```go
   // pkg/serviceaccount/jwt.go
   func (j *jwtTokenAuthenticator) AuthenticateToken(ctx context.Context, tokenData string) (*authenticator.Response, bool, error) {
   	// 1.校验token格式正确
   	if !j.hasCorrectIssuer(tokenData) {
   		return nil, false, nil
   	}
   
   	// 2.解析JWT对象
   	tok, err := jwt.ParseSigned(tokenData)
   	...
   
   	public := &jwt.Claims{}
   	private := j.validator.NewPrivateClaims()
   
   	// TODO: Pick the key that has the same key ID as `tok`, if one exists.
   	var (
   		found   bool
   		errlist []error
   	)
   	// 3.使用--service-account-key-file提供的密钥，反序列化JWT
   	for _, key := range j.keys {
   		if err := tok.Claims(key, public, private); err != nil {
   			errlist = append(errlist, err)
   			continue
   		}
   		found = true
   		break
   	}
   
   	...
       
   	// 4.验证namespace是否正确、serviceAccountName、serviceAccountID是否存在，token是否失效
   	sa, err := j.validator.Validate(ctx, tokenData, public, private)
   	if err != nil {
   		return nil, false, err
   	}
   
   	return &authenticator.Response{
   		User:      sa.UserInfo(),
   		Audiences: auds,
   	}, true, nil
   }
   ```

   服务账号被身份认证后，所确定的用户名为 `system:serviceaccount:<NAMESPACE>:<SERVICEACCOUNT>`， 并被分配到用户组 `system:serviceaccounts` 和 `system:serviceaccounts:<NAMESPACE>`。


### 2.1.5 OpenID Connect Token
OpenID Connect Token(OIDC) 是一套基于 OAuth2.0 协议的轻量级认证规范，其提供了通过 API 进行身份交互的框架。OIDC 认证除了认证请求外，还会标明请求的用户身份（ID Token）。其中 Token 被称为 ID Token，此 ID Token 是 JWT，具有服务器签名的相关字段。认证流程如下：
1. 用户想要访问 kube-apiserver，先通过认证服务（Auth Service，例如 Google Accounts 服务）认证自己，得到access_token、id_token 和 refresh_token。
2. 用户把 access_token、id_token 和 refresh_token 配置到客户端应用程序，例如：kubectl 或者 dashboard 工具
3. 客户端使用 Token 以用户身份访问 kube-apiserver

kube-apiserver 和 Auth Service 没有直接交互，而是鉴定客户端发送过来的 Token 是否合法。完整的 OIDC 认证过程如下图所示：

{{< mermaid >}}
sequenceDiagram
    participant user as 用户
    participant idp as 身份提供者 
    participant kube as Kubectl
    participant api as API 服务器

    user ->> idp: 1. 登录到 IdP
    activate idp
    idp -->> user: 2. 提供 access_token,<br>id_token, 和 refresh_token
    deactivate idp
    activate user
    user ->> kube: 3. 调用 Kubectl 并<br>设置 --token 为 id_token<br>或者将令牌添加到 .kube/config
    deactivate user
    activate kube
    kube ->> api: 4. Authorization: Bearer...
    deactivate kube
    activate api
    api ->> api: 5. JWT 签名合法么？
    api ->> api: 6. JWT 是否已过期？(iat+exp)
    api ->> api: 7. 用户被授权了么？
    api -->> kube: 8. 已授权：执行<br>操作并返回结果
    deactivate api
    activate kube
    kube --x user: 9. 返回结果
    deactivate kube
{{< /mermaid >}}

1. 登录到身份服务（即 Auth Server）
2. 身份服务将为你提供 access_token、id_token 和 refresh_token
3. 用户在使用 kubectl 时，将 id_token 设置为 `--token` 标志值，或者将其直接添加到 kubeconfig 中
4. kubectl 将 id_token 设置为 Authorization 的请求头，发送给 API 服务器
5. API 服务器将负责通过检查配置中引用的证书来确认 JWT 的签名是合法的
6. 检查确认 id_token 尚未过期
7. 确认用户有权限执行操作
8. 鉴权成功之后，API 服务器向 kubectl 返回响应
9. kubectl 向用户提供反馈信息

kube-apiserver 不与 Auth Service 交互就可以认证 Token 的合法性，关键在于第 5 步，所有 JWT 都由颁发给它的 Auth Service进行了数字签名，只需要在 kube-apiserver 的启动参数中，配置信任的 Auth Server 证书，用它来验证 id_token 是否合法。

1. 启用

- `--oidc-ca-file`：指向一个 CA 证书的路径，该 CA 负责对你的身份服务的 Web 证书提供签名。默认值为宿主系统的根 CA（`/etc/kubernetes/ssl/kc-ca.pem`）。
- `--oidc-client-id`：所有令牌都应发放给此客户 ID。
- `--oidc-groups-claim`：JWT 声明的用户组名称。
- `--oidc-groups-prefix`：添加到组申领的前缀，用来避免与现有用户组名（如：`system:` 组）发生冲突。例如，此标志值为 `oidc:` 时，所得到的用户组名形如 `oidc:engineering` 和 `oidc:infra`。 
- `--oidc-issuer-url`：允许 API 服务器发现公开的签名密钥的服务的 URL。只接受模式为 `https://` 的 URL。此值通常设置为服务的发现 URL，不含路径。例如："https://accounts.google.com" 或 "https://login.salesforce.com"。此 URL 应指向 .well-known/openid-configuration 下一层的路径。 
- `--oidc-required-claim`：取值为一个 key=value 偶对，意为 ID 令牌中必须存在的申领。如果设置了此标志，则 ID 令牌会被检查以确定是否包含取值匹配的申领。此标志可多次重复，以指定多个申领。
- `--oidc-username-claim`：JWT 声明的用户名称。默认情况下使用 `sub` 值，即最终用户的一个唯一的标识符。
- `--oidc-username-prefix`：要添加到用户名申领之前的前缀，用来避免与现有用户名（例如：`system:` 用户）发生冲突。

2. 认证实现

```go
// staging/src/k8s.io/apiserver/plugin/pkg/authenticator/token/oidc/oidc.go
func (a *Authenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
	...
	idToken, err := verifier.Verify(ctx, token)
	...
	return &authenticator.Response{User: info}, true, nil
}
```

整个认证逻辑，大体上是解析 id_token，把其中的 user、group 信息取出，组成 User 对象返回。在返回之前，要对针对 kube-apiserver 的各个 reqiured_claims 入参校验，看看从 id_token 中的值是否匹配。

### 2.1.6 Webhook Token

webhook 也被称为钩子，是一种基于 HTTP 协议的回调机制，当客户端发送的认证请求到达 kube-apiserver 时，kubbe-apiserver 回调钩子方法，将验证信息发送给远程的 webhook 服务器进行验证，让根据返回的状态码判断是否认证通过。

1. 启用
- `--authentication-token-webhook-config-file`：指向一个配置文件，其中描述 如何访问远程的 Webhook 服务。
- `--authentication-token-webhook-cache-ttl`：用来设定身份认证决定的缓存时间。默认时长为 2 分钟。

配置文件使用 kubeconfig 文件的格式。文件中，clusters 指代远程服务，users 指代远程 API 服务 Webhook。下面是一个例子：

```yaml
# Kubernetes API 版本
apiVersion: v1
# API 对象类别
kind: Config
# clusters 指代远程服务
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # 用来验证远程服务的 CA
      server: https://authn.example.com/authenticate # 要查询的远程服务 URL。必须使用 'https'。

# users 指代 API 服务的 Webhook 配置
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # Webhook 插件要使用的证书
      client-key: /path/to/key.pem          # 与证书匹配的密钥

# kubeconfig 文件需要一个上下文（Context），此上下文用于本 API 服务器
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-sever
  name: webhook
```

当客户端尝试在 API 服务器上使用持有者令牌完成身份认证（ 如前所述）时， 身份认证 Webhook 会用 POST 请求发送一个 JSON 序列化的对象到远程服务。 该对象是 `authentication.k8s.io/v1beta1` 组的 `TokenReview` 对象， 其中包含持有者令牌。 Kubernetes 不会强制请求提供此 HTTP 头部。

要注意的是，Webhook API 对象和其他 Kubernetes API 对象一样，也要受到同一 版本兼容规则约束。 实现者要了解对 Beta 阶段对象的兼容性承诺，并检查请求的 apiVersion 字段， 以确保数据结构能够正常反序列化解析。此外，API 服务器必须启用 `authentication.k8s.io/v1beta1` API 扩展组 （`--runtime-config=authentication.k8s.io/v1beta1=true`）。

2. 认证实现
```go
// staging/src/k8s.io/apiserver/pkg/authentication/token/cache/cached_token_authenticator.go
func (a *cachedTokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
	record := a.doAuthenticateToken(ctx, token)
	if !record.ok || record.err != nil {
		return nil, false, record.err
	}
	for key, value := range record.annotations {
		audit.AddAuditAnnotation(ctx, key, value)
	}
	return record.resp, true, nil
}
```
`a.doAuthenticateToken(ctx, token)` 是认证过程的核心，首先从缓存中查找是否已认证，有则直接返回，没有调用远程 webhook 服务验证。
```go
// staging/src/k8s.io/apiserver/plugin/pkg/authenticator/token/webhook/webhook.go
func (w *WebhookTokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
    ...
    
    webhook.WithExponentialBackoff(ctx, w.initialBackoff, func() error {
		result, err = w.tokenReview.Create(ctx, r, metav1.CreateOptions{})
		return err
	}, webhook.DefaultShouldRetry)
    
    ...
    
    if !r.Status.Authenticated {
		var err error
		if len(r.Status.Error) != 0 {
			err = errors.New(r.Status.Error)
		}
		return nil, false, err
	}
    
    ...
    
    return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   r.Status.User.Username,
			UID:    r.Status.User.UID,
			Groups: r.Status.User.Groups,
			Extra:  extra,
		},
		Audiences: auds,
	}, true, nil
}
```

通过 w.tokenReview.Create 发送 POST 请求到远程 webhook 服务，并在 body 体中携带认真信息，根据返回值 Status.Authenticated 判断是否认证通过。

### 2.1.7 Authentication Proxy

API 服务器可以配置成从请求的头部字段值（如 X-Remote-User）中辩识用户。这一设计是用来与某身份认证代理一起使用 API 服务器，代理负责设置请求的头部字段值。

认证代理有几个列表，
- 用户名列表：建议设置为 "X-Remote-User"。必选
- 组列表：建议设置为 "X-Remote-Group"。可选
- 额外列表：建议设置为 "X-Remote-Extra-"。可选

1. 启用
- `--requestheader-client-ca-file`：指定有效的客户端 CA 证书。
- `--requestheader-allowed-names`：指定通用名称（Common Name）
- `--requestheader-username-headers`：指定用户名列表
- `--requestheader-group-headers`：指定组名列表
- `--requestheader-extra-headers-prefix`：指定额外列表

2. 认证

```go
// staging/src/k8s.io/apiserver/pkg/authentication/request/headerrequest/requestheader.go
func (a *requestHeaderAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	// 用户信息
	name := headerValue(req.Header, a.nameHeaders.Value())
	if len(name) == 0 {
		return nil, false, nil
	}
	// 组信息
	groups := allHeaderValues(req.Header, a.groupHeaders.Value())
	// 额外信息
	extra := newExtra(req.Header, a.extraHeaderPrefixes.Value())

	...

	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   name,
			Groups: groups,
			Extra:  extra,
		},
	}, true, nil
}
```
在进行认证代理认证时，requestHeader就是实现方式，分别从 HTTP Header 读出用户、组和额外信息，返回给客户端。


## 2.2 其他

> 有关匿名请求、用户伪装和 client-go 插件代理，请移步官网：[用户认证](http://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#anonymous-requests)