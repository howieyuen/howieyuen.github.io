---
author: Yuan Hao
date: 2020-01-24
title: 深入了解 Service
tag: [Service]
---

# 1. 基本概念

## 1.1 Service 定义详解

Service 是对一组提供相同功能的 Pods 的抽象，并为它们提供一个统一的入口。借助 Service，
应用可以方便的实现服务发现与负载均衡，并实现应用的零宕机升级。Service 通过标签来选取服务后端，
一般配合 Replication Controller 或者 Deployment 来保证后端容器的正常运行。
这些匹配标签的 Pod IP 和端口列表组成 endpoints，
由 kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: string
  namespace: string
  labels:
  - name: string
  annotations:
  - name: string
spec:
  selector: []
  # ClusterIP、NodePort、LoadBalancer
  type: string
  # type=ClusterIP, 有自动分配的能力；type=LoadBalancer，需指定            
  clusterIP: string    
  # 是否支持session，默认为空，可选值ClutserIP，同一个client的request，都发送到同一个后端Pod   
  sessionAffinity: string 
  ports:
  - name: string
    # tcp、udp，默认tcp
    protocol: string      
    port: int
    targetPort: int
    nodePort: int
# spec.type=LoadBalancer,设置外部负载均衡器地址，用于公有云环境
status:                   
  loadBalancer:
    ingress:
      ip: string
        hostname: string
```

## 1.2 Service 分类

- ClusterIP：默认类型，自动分配一个仅 cluster 内部可以访问的虚拟 IP
- NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，
  这样就可以通过 `http://<NodeIP>:NodePort` 来访问该服务。
  如果 kube-proxy 设置了 `--nodeport-addresses=10.240.0.0/16`（v1.10 支持），
  那么仅该 NodePort 仅对设置在范围内的 IP 有效。
- LoadBalancer：在 NodePort 的基础上，借助 cloud provider 创建一个外部的负载均衡器，
  并将请求转发到 `<NodeIP>:NodePort`
- ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 `spec.externlName` 设定）。
  需要 kube-dns 版本在 1.7 以上。

# 2. Service 基本用法

一般来说，对外提供服务的应用程序需要通过某种机制来实现，
对于容器应用最简便的方式就是通过 TCP/IP 机制及监听IP和端口号来实现。

直接通过Pod的IP地址和端口号可以访问到容器应用内的服务，但是 Pod 的 IP 地址是不可靠的，
例如当Pod所在的Node发生故障时，Pod 将被 Kubernetes 重新调度到另一个 Node，
Pod 的 IP 地址将发生变化。
更重要的是，如果容器应用本身是分布式的部署方式，通过多个实例共同提供服务，
就需要在这些实例的前端设置一个负载均衡器来实现请求的分发。
Kubernetes中的Service就是用于解决这些问题的核心组件。

Service 定义中的关键字段是 ports 和 selector。通过 selector 与 Pod 关联，
port 描述 service 本身的端口，targetPort 表示流量转发的端口，也就是 Pod 的端口，
从而完成访问 service 负载均衡到后端任意一个 Pod。Kubernetes提供了两种负载分发策略：
RoundRobin 和 SessionAffinity，具体说明如下：
- RoundRobin：轮询模式，即轮询将请求转发到后端的各个 Pod 上。
- SessionAffinity：基于客户端IP地址进行会话保持的模式，
  即第 1 次将某个客户端发起的请求转发到后端的某个 Pod 上，
  之后从相同的客户端发起的请求都将被转发到后端相同的 Pod 上。

在默认情况下，Kubernetes 采用 RoundRobin 模式对客户端请求进行负载分发，
但我们也可以通过设置 `service.spec.sessionAffinity=ClientIP` 来启用 SessionAffinity 策略。
这样，同一个客户端IP发来的请求就会被转发到后端固定的某个Pod上了。

## 2.1 集群内访问集群外服务

到现在为止，我们己经讨论了后端是集群中运行的一个或多个 Pod 的服务。
但也存在希望通过 Kubernetes 服务特性暴露外部服务的情况。
不要让服务将连接重定向到集群中的Pod，而是让它重定向到外部 IP 和端口。
这样做可以让你充分利用服务负载平衡和服务发现。
在集群中运行的客户端 Pod 可以像连接到内部服务一样连接到外部服务。

首先要知道，service 和 Pod 并不是直接相连的，此二者之间还有一个对象叫 endpoint。
service 根据 selector 找到后端 Pod，用 Pod IP 和端口创建与 service 同名的 endpoint，
记录 Pod IP。当 Pod 异常被删除重建后，获得的新地址，只需要更新 endpoint 中记录的 Pod I P即可。
因此，想要访问集群外部的服务，可手动配置 service 的 endpoint。

### 2.1.1 创建没有 selector 的 Service

1. 创建没有 selector 的 Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

2. 为没有选择器的服务创建 Endpoint 资源
```yaml
apiVersion: v1
kind: Endpoint
metadata:
  # endpoint的名称必须和服务的名称相匹配
  name: external-service  
subsets:
  - addresses:
    # 将service重定向到endpoint的地址
    - ip: 11.11.11.11      
    - ip: 22.22.22.22
    ports:
    # endpoint目标端口
    - port: 80             
```

Endpoint 对象需要与服务具有相同的名称，并包含该服务的目标 IP 地址和端口列表。
服务和 Endpoint 资源都发布到服务器后，这样服务就可以像具有 Pod 选择器那样的服务正常使用。
在服务创建后创建的容器将包含服务的环境变量，并且与其 IP:Port 对的所有连接都将在服务端点之间进行负载均衡。

### 2.1.2 创建 ExternalName 的 service

除了手动配置服务的 Endpoint 来代替公开外部服务方法，有一种更简单的方法，
就是通过其完全限定域名(FQDN)访问外部服务。

要创建一个具有别名的外部服务的服务时，要将创建服务资源的一个 type 字段设置为 ExternalName。
例如，设想一下在 api.somecompany.com 上有公共可用的 API 可以定义一个指向它的服务，
如下面的代码清单所示：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: someapi.somecompany.com
  ports:
  - port: 80
```

服务创建完成后，Pod 可以通过 external-service.default.svc.cluster.local 域名
(甚至是 external-service)连接到外部服务，而不是使用服务的实际 FQDN。
这隐藏了实际的服务名称及其使用该服务的 Pod 的位置，允许修改服务定义，
并且在以后如果将其指向不同的服务，只需简单地修改 externalName 属性，
或者将类型重新变回 Cluster IP并为服务创建 Endpoint ——无论是手动创建，
还是对服务上指定标签选择器使其自动创建。

ExternalName 服务仅在DNS级别实施——为服务创建了简单的 CNAME DNS 记录。
因此，连接到服务的客户端将直接连接到外部服务，完全绕过服务代理。
出于这个原因，这些类型的服务甚至不会获得集群IP。

## 2.2 集群外访问集群内服务

### 2.2.1 NodePort 服务

将服务的类型设置成 `NodePort`：每个集群节点都会在节点上打开一个端口，对于 NodePort 服务，
每个集群节点在节点本身(因此得名叫 NodePort)上打开一个端口，
并将在该端口上接收到的流量重定向到基础服务。该服务仅在内部集群 IP 和端口上才可访间，
但也可通过所有节点上的专用端口访问。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: kubia
```

### 2.2.2 LoadBalancer 服务

在云提供商上运行的 Kubernetes 集群通常支持从云基础架构自动提供负载平衡器。
所有需要做的就是设置服务的类型为 LoadBadancer 而不是 NodePort。
负载均衡器拥有自己独一无二的可公开访问的IP地址，并将所有连接重定向到服务。
可以通过负载均衡器的 IP 地址访问服务。

如果 Kubemetes 在不支持 LoadBadancer 服务的环境中运行，则不会调配负载平衡器，
但该服务仍将表现得像一个 NodePort 服务。这是因为 LoadBadancer 服务是 NodePort 服务的扩展。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
status:
  loadBalancer:
    ingress:
    - ip: 146.148.47.155
```

# 3. 通过 Ingress 暴露服务

- 为什么需要 Ingress？

一个重要的原因是每个 LoadBalancer 服务都需要自己的负载均衡器，以及独有的公有 IP 地址，
而 Ingress 只需要一个公网 IP 就能为许多服务提供访问。当客户端向 Ingress 发送 HTTP 请求时，
Ingress 会根据请求的主机名和路径决定请求转发到的服务。

## 3.1 创建 Ingress Controller 和默认的 backend 服务

在定义 Ingress 策略之前，需要先部署 Ingress Controller，
以实现为所有后端 Service 都提供一个统一的入口。
Ingress Controller 需要实现基于不同 HTTP URL 向后转发的负载分发规则，
并可以灵活设置 7 层负载分发策略。如果公有云服务商能够提供该类型的 HTTP 路由 LoadBalancer，
则也可设置其为 Ingress Controller。

在 Kubernetes 中，Ingress Controller 将以 Pod 的形式运行，
监控 API Server 的 ingress 接口后端的 backend services，
如果 Service 发生变化，则 Ingress Controller 应自动更新其转发规则。

下面的例子使用Nginx来实现一个 Ingress Controller，需要实现的基本逻辑如下：
1. 监听 API Server，获取全部 Ingress 的定义。
2. 基于 Ingress 的定义，生成 Nginx 所需的配置文件 /etc/nginx/nginx.conf。
3. 执行 nginx -s reload 命令，重新加载 nginx.conf 配置文件的内容。

为了让 Ingress Controller 正常启动，还需要为它配置一个默认的 backend，
用于在客户端访问的 URL 地址不存在时，返回一个正确的 404 应答。
这个 backend 服务用任何应用实现都可以，只要满足对根路径“/”的访问返回 404 应答，
并且提供 /healthz 路径以使 kubelet 完成对它的健康检查。
另外，由于 Nginx 通过 default-backend-service 的服务名称（Service Name）去访问它，
所以需要 DNS 服务正确运行。

## 3.2 创建 Ingress 资源

### 3.2.1 转发到单个后端服务上

基于这种设置，客户端到 Ingress Controller 的访问请求都将被转发到后端的唯一 Service 上，
在这种情况下 Ingress 无须定义任何 rule。

通过如下所示的设置，对 Ingress Controller 的访问请求都将被转发到“myweb:8080”这个服务上。
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: webapp
    servicePort: 8080
```

### 3.2.2 将不同的服务映射到相同主机的不同路径

这种配置常用于一个网站通过不同的路径提供不同的服务的场景，
例如 /web 表示访问 Web 页面，/api表示访问 API 接口，对应到后端的两个服务，
通过Ingress的设置很容易就能将基于 URL 路径的转发规则定义出来。

通过如下所示的设置，对 mywebsite.com/web 的访问请求将被转发到 web-service:80 服务上；
对 mywebsite.com/api 的访问请求将被转发到 api-service:80 服务上：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec
  rules:
  - host mywebsite.com
    http:
      paths:
      - path: /web
        backend:
          serviceName: web-service
          servicePort: 80
      - path: /api
        backend:
          serviceName: api-service
          servicePort: 80
```

### 3.2.3 不同的域名(虚拟主机名)被转发到不同的服务上

这种配置常用于一个网站通过不同的域名或虚拟主机名提供不同服务的场景，
例如 foo.example.com 域名由 foo 提供服务，bar.example.com 域名由 bar 提供服务。

通过如下所示的设置，对“foo.example.com”的访问请求将被转发到“foo:80”服务上，
对“bar.example.com”的访问请求将被转发到“bar:80”服务上：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: I
        backend:
          serviceName: foo
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
        - path: I
          backend:
            serviceName: bar
            servicePort: 80
```

### 3.2.4 不使用域名的转发规则

这种配置用于一个网站不使用域名直接提供服务的场景，
此时通过任意一台运行 ingress-controller 的 Node 都能访问到后端的服务。

下面的配置为将“<ingresscontroller-ip>/demo”的访问请求转发到“webapp:8080/demo”服务上：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  rules:
  - http:
      paths:
      - path: /demo
        backend:
          serviceName: webapp
          servicePort: 8080
```

注意，使用无域名的 Ingress 转发规则时，将默认禁用非安全 HTTP，强制启用 HTTPS。
可以在 Ingress 的定义中设置一个 annotation[ingress.kubernetes.io/ssl-redirect: "false"]
来关闭强制启用 HTTPS 的设置：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /demo
        backend:
          serviceName: webapp
          servicePort: 8080
```

## 3.3 Ingress 的 TLS 安全设置

当客户端创建到 Ingress 控制器的 TLS 连接时，控制器将终止 TLS 连接。
客户端和控制器之间的通信是加密的，而控制器和后端 Pod 之间的通信则不是运行在 Pod 上的应用程序不需要支持 TLS。
例如，如果 Pod 运行 web 服务器，则它只能接收 HTTP 通信，并让 Ingress 控制器负责处理与 TLS 相关的所有内容。
要使控制器能够这样做，需要将证书和私钥附加到 Ingress。
这两个必需资源存储在称为 Secret 的 Kubernetes 资源中，然后在 Ingress manifest 中引用它。

为了 Ingress 提供 HTTPS 的安全访问，可以为 Ingress 中的域名进行 TLS 安全证书的设置。设置的步骤如下。

1. 创建自签名的密钥和 SSL 证书文件
```bash
openssl genrsa -out tls.key 2048
openssl req -new - x509 -key tls.key -out tls.cert -days 360 -subject/CN=kubia.example.com
```
2. 将证书保存到Kubernetes中的一个Secret资源对象上
```bash
kubectl create secret tls mywebsite-tls-secret --cert=tls.cert --key=tls.key
```

3. 将该 Secret 对象设置到 Ingress 中
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mywebsite-ingress-tls
spec:
  tls:
  - hosts:
    - mywebsite.com
    secretName: mywebsite-tls-secret
  rules:
  - http:
      paths:
      - path: /demo
        backend:
          serviceName: webapp
          servicePort: 8080
```

根据提供服务的网站域名是一个还是多个，可以使用不同的操作完成前两步 SSL 证书和 Secret 对象的创建，
在只有一个域名的情况下设置相对简单。第3步对于这两种场景来说是相同的。