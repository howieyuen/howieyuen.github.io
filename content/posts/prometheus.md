---
title: Prometheus Operator 设计文档
date: 2022-9-28
tags: [Prometheus]
categories: [Design Doc]
---

> 原文链接：[https://prometheus-operator.dev/docs/operator/design/](https://prometheus-operator.dev/docs/operator/design/)

本文将阐述 Prometheus Operator 管理的 CRD 之间的设计和交互。由 Prometheus Operator 管理的 CRD 有：
- [Prometheus](https://prometheus-operator.dev/docs/operator/design/#prometheus)
- [Alertmanager](https://prometheus-operator.dev/docs/operator/design/#alertmanager)
- [ThanosRuler](https://prometheus-operator.dev/docs/operator/design/#thanosruler)
- [ServiceMonitor](https://prometheus-operator.dev/docs/operator/design/#servicemonitor)
- [PodMonitor](https://prometheus-operator.dev/docs/operator/design/#podmonitor)
- [Probe](https://prometheus-operator.dev/docs/operator/design/#probe)
- [PrometheusRule](https://prometheus-operator.dev/docs/operator/design/#prometheusrule)
- [AlertmanagerConfig](https://prometheus-operator.dev/docs/operator/design/#alertmanagerconfig)

<!--more-->

## Prometheus

`Prometheus`(CRD) 以声明方式定义所需的 [Prometheus](https://prometheus.io/docs/prometheus)，设置在 Kubernetes 集群中运行。
它提供了选项来配置已部署的 Prometheus 实例的副本数量、持久存储和向其发送警报的 Alertmanagers。

对于每个 `Prometheus` 资源，Operator 在同一个命名空间中部署一个或多个 `StatefulSet` 对象（statefulset 的数量等于 shard 的数量，但默认为 1）。
CRD 通过标签和命名空间选择器定义哪些 `ServiceMonitor`、`PodMonitor` 和 `Probe` 对象应该与部署的 Prometheus 实例相关联。

CRD 还定义了应该调和哪些 `PrometheusRules` 对象。Operator 不断地协调 CR 并生成一个或多个包含 Prometheus 配置的 `Secret` 对象。
在 Prometheus Pod 中作为 sidecar 运行的 config-reloader 容器检测到配置的任何更改，并在需要时重新加载 `Prometheus`。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: true # Expose the Prometheus Admin API
```

## Alertmanager

`Alertmanager`(CRD) 以声明方式定义所需的 [Alertmanager](https://prometheus.io/docs/alerting)
设置以在 Kubernetes 集群中运行。它提供了配置副本数量和持久化存储的选项。

对于每个 `Alertmanager` 资源，Operator 在同一个命名空间中部署一个 `StatefulSet`。
Alertmanager Pod 被配置为挂载一个名为 `alertmanager-<alertmanager-name>` 的 `Secret`，
该 `Secret` 在 key 为 `alertmanager.yaml` 下保存 Alertmanager 配置。

当有两个或更多配置的副本时，Operator 以高可用模式运行 Alertmanager 实例。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: main
spec:
  replicas: 3
  resources:
    requests:
      memory: 400Mi
```

## ThanosRuler

`ThanosRuler` (CRD) 以声明方式定义了所需的 [Thanos Ruler](https://github.com/thanos-io/thanos/blob/main/docs/components/rule.md)
设置以在 Kubernetes 集群中运行。借助 ThanosRuler，可以跨多个 Prometheus 实例处理记录和警报规则。

一个 ThanosRuler 实例至少需要一个查询端点，该端点指向 Thanos Queriers 或 Prometheus 实例的位置。
更多信息详见 [Thanos](https://prometheus-operator.dev/docs/operator/thanos/) 小节 。

## ServiceMonitor

`ServiceMonitor`(CRD) 以声明方式定义应如何监视一组动态服务。
使用标签选择器来定义哪些 `Service` 被选择以使用所需的配置进行监视。
这允许组织引入关于如何公开指标的约定，然后按照这些约定自动发现新服务，而无需重新配置系统。

为了让 Prometheus 监控 Kubernetes 中的任何应用程序，需要 `Endpoints` 对象。
`Endpoints` 本质上是 IP 地址列表。通常，`Endpoints` 由 `Service` 填充。
`Service` 通过标签选择器发现 `Pod` 并将其添加到 `Endpoints` 中。

一个 `Service` 可能会暴露一个或多个服务端口，这些端口通常由指向 `Pod` 的多个端点列表支持。
这也反映在各自的 `Endpoints` 对象中。Prometheus Operator 引入的 `ServiceMonitor`
对象依次发现那些 `Endpoints` 对象并配置 `Prometheus` 来监控这些 `Pod`。

`ServiceMonitorSpec` 的 `endpoints` 部分，用于配置这些端点的哪些端口将被抓取以获取指标，以及使用哪些参数。
对于高级用例，可能需要监控不直接属于 `Service` 的 `Endpoints` 所支持 `Pod` 的端口。
因此，在 `endpoints` 部分中指定端点时，严格使用它们。

{{< hint info >}}
注意：`endpoints`（小写）是 `ServiceMonitor` CRD 中的字段，而 `Endpoints`（大写）是 Kubernetes 对象类型。
{{< /hint >}}

`ServiceMonitors` 和发现的目标都可能来自任何命名空间。这对于允许跨命名空间监控用例很重要，例如，用于元监控。
使用 `PrometheusSpec` 的 `ServiceMonitorNamespaceSelector`，
可以限制各个 Prometheus 服务器从中选择 `ServiceMonitor` 的命名空间。
使用 `ServiceMonitorSpec` 的 `namespaceSelector`，可以限制允许从中发现 `Endpoints` 对象的命名空间。

可以像这样在所有命名空间中发现目标：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
  namespaceSelector:
    any: true
```

## PodMonitor

`PodMonitor` (CRD) 允许以声明方式定义应如何监视一组动态 `Pod`。使用标签选择来定义选择要使用所需配置监视哪些 `Pod`。
这允许组织引入关于如何公开指标的约定，然后按照这些约定自动发现新的 `Pod`，而无需重新配置系统。

`Pod` 是一个或多个容器的集合，可以在多个端口上公开 Prometheus 指标。

Prometheus Operator 引入的 `PodMonitor` 对象会发现这些 `Pod`，并为 Prometheus 服务器生成相关配置，以便对其进行监控。

`PodMonitorSpec` 的 `PodMetricsEndpoints` 部分用于配置 `Pod` 的哪些端口将被抓取以获取指标，以及使用哪些参数。

`PodMonitor` 和发现的目标都可能来自任何命名空间。这对于允许跨命名空间监控用例很重要，例如用于元监控。
使用 `PodMonitorSpec` 的 `namespaceSelector`，可以限制允许发现 `Pod` 的命名空间。

一旦可以像这样发现所有命名空间中的目标：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: example-app
spec:
  selector:
    matchLabels:
      app: example-app
  podMetricsEndpoints:
  - port: web
  namespaceSelector:
    any: true
```

## Probe

`Probe` (CRD) 允许以声明方式定义应如何监视 `Ingress` 和静态目标。
除了目标之外，`Probe` 对象还需要一个探测器，它监控目标并为 Prometheus 提供指标以进行抓取的服务。
通常，这是使用 [black exporter](https://github.com/prometheus/blackbox_exporter) 实现的。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  labels:
    group: group1
  name: testprobe
  namespace: default
spec:
  module: http_2xx
  prober:
    path: /probe
    scheme: http
    url: blackbox.exporter.io
  targets:
    ingress:
      relabelingConfigs:
      - action: replace
        replacement: bar
        targetLabel: foo
      namespaceSelector:
        any: true
      selector:
        matchLabels:
          prometheus.io/probe: 'true'
```

## PrometheusRule

`PrometheusRule`(CRD) 以声明方式定义 Prometheus 或 Thanos Ruler 实例使用的所需 Prometheus 规则。
警报和记录规则由 Operator 协调并动态加载，无需重新启动 Prometheus/Thanos Ruler。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: etcd-rules
  namespace: monitoring
spec:
  groups:
  - name: etcd
    rules:
    - alert: EtcdClusterUnavailable
      annotations:
        summary: etcd cluster small
        description: If one more etcd peer goes down the cluster will be unavailable
      expr: |
        count(up{job="etcd"} == 0) > (count(up{job="etcd"}) / 2 - 1)
      for: 3m
      labels:
        severity: critical
```

## AlertmanagerConfig

`AlertmanagerConfig` (CRD) 以声明方式指定 Alertmanager 配置的子部分，允许将警报路由到自定义接收器，并设置禁止规则。 
`AlertmanagerConfig` 可以在命名空间级别上定义，为 Alertmanager 提供聚合配置。下面提供了如何使用它的示例。请注意，这个 CRD 还不稳定。

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: config-example
  labels:
    alertmanagerConfig: example
spec:
  route:
    groupBy: ['job']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 12h
    receiver: 'wechat-example'
  receivers:
  - name: 'wechat-example'
    wechatConfigs:
    - apiURL: 'http://wechatserver:8080/'
      corpID: 'wechat-corpid'
      apiSecret:
        name: 'wechat-config'
        key: 'apiSecret'
---
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: wechat-config
data:
  apiSecret: d2VjaGF0LXNlY3JldAo=
```
