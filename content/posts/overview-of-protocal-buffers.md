---
title: Protocal Buffers 概述
date: 2022-05-05
categories: [protobuf]
---

> 原文链接：[Overview | Protocol Buffers | Google Developers](https://developers.google.com/protocol-buffers/docs/overview)

协议缓冲区（Protocol Buffers）提供了一种语言中立、平台中立、可扩展的机制，用来序列化结构化数据，并且支持向前/向后兼容。
它类似于 JSON，只是它更小更快，并且能生成本地语言绑定。

协议缓冲区包含以下模块：
- 定义语言（在 `.proto` 文件中创建）
- 连接数据的代码（ proto 编译器生成）
- 特定语言的运行时库
- 序列化格式的数据（写入文件或者通过网络传输）

<!--more-->

## Protocol Buffers 解决了哪些问题？{#solve}

协议缓冲区为大小高达几 MB 的类型化、结构化数据包提供了一种序列化格式。
该格式既适用于临时网络流量，又适用于长期数据存储。
可以使用新信息扩展协议缓冲区，而无需废弃当前数据或更新代码。

协议缓冲区是 Google 最常用的数据格式。
它们广泛用于服务器间通信以及磁盘上数据的归档存储。
Protocol buffer 的 `message` 和 `service` 由工程师编写的 `.proto` 文件描述。
下面是一个 `message` 示例：

```protobuf
message Person {
  optional string name = 1;
  optional int32 id = 2;
  optional string email = 3;
}
```

proto 编译器在构建 `.proto` 文件时，生成各种编程语言的代码
（在后文的<a href= "#cross-lang">跨语言兼容性</a>中有说明）来操作相应的协议缓冲区。
每个生成的类都包含每个字段和方法的简单访问器，用于序列化和解析整个结构与原始字节之间的关系。
下面向你展示了一个使用这些生成方法的示例：

```protobuf
Person john = Person.newBuilder()
    .setId(1234)
    .setName("John Doe")
    .setEmail("jdoe@example.com")
    .build();
output = new FileOutputStream(args[0]);
john.writeTo(output);
```

由于协议缓冲区在 Google 的各种服务中广泛使用，并且其中的数据可能会保留一段时间，因此保持向后兼容性至关重要。
协议缓冲区允许无缝支持对任何协议缓冲区的更改，包括添加新字段和删除现有字段，而不会破坏现有服务。
有关此话题的更多信息，请参阅后文的<a href="#update-defs">在不更新代码的情况下更新原型定义</a>。

## Protocol Buffers 带来的好处是什么？{#benefits}

协议缓冲区非常适合需要以语言中立、平台中立并且可扩展的方式去序列化结构化、类似记录、类型化数据的情况。
它们最常用于定义通信协议（与 gRPC 一起）和数据存储。

使用协议缓冲区的优势包括：
- 紧凑的数据存储
- 快速解析
- 多编程语言的可用性
- 通过自动生成类的优化功能

### 跨语言兼容性 {#cross-lang}

以任何受支持的编程语言编写的代码都可以读取相同的消息。
你可以让一个平台上的 Java 程序从一个软件系统捕获数据，根据 `.proto` 定义对其进行序列化，
然后在另一个平台上运行的单独 Python 应用程序中从序列化数据中提取特定值。

协议缓冲区编译器（protoc）直接支持以下语言：

- [C++](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated#invocation)
- [C#](https://developers.google.com/protocol-buffers/docs/reference/csharp-generated#invocation)
- [Java](https://developers.google.com/protocol-buffers/docs/reference/java-generated#invocation)
- [Kotlin](https://developers.google.com/protocol-buffers/docs/reference/kotlin-generated#invocation)
- [Objective-C](https://developers.google.com/protocol-buffers/docs/reference/objective-c-generated#invocation)
- [PHP](https://developers.google.com/protocol-buffers/docs/reference/php-generated#invocation)
- [Python](https://developers.google.com/protocol-buffers/docs/reference/python-generated#invocation)
- [Ruby](https://developers.google.com/protocol-buffers/docs/reference/ruby-generated#invocation)

Google 支持以下语言，但项目的源代码位于 GitHub 存储库中。
protoc 编译器使用这些语言的插件：
- [Dart](https://github.com/google/protobuf.dart)
- [Go](https://github.com/protocolbuffers/protobuf-go)

其他语言不直接由 Google 支持，而是由其他 GitHub 项目支持。
这些语言包含在
[Protocol Buffers 的第三方插件](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)中。

### 跨项目支持 {#cross-proj}

你可以通过在驻留在特定项目代码库之外的 `.proto` 文件中定义 `message` 类型来跨项目使用协议缓冲区。
如果你正在定义预计在你的直接团队之外广泛使用的 `message` 类型或枚举，你可以将它们放在自己的文件中，而无需依赖。

在 Google 中广泛使用的几个原型定义示例是
[timestamp.proto](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/timestamp.proto)
和 [status.proto](https://github.com/googleapis/googleapis/blob/master/google/rpc/status.proto)。

### 在不更新代码的情况下更新原型定义 {#update-defs}

软件产品向后兼容是标准，但向前兼容却不太常见。
只要你在更新 `.proto` 定义时遵循一些[简单的做法](https://developers.google.com/protocol-buffers/docs/proto#updating)，
旧代码将毫无问题地读取新消息，而忽略任何新添加的字段。
对于旧代码，已删除的字段将具有默认值，已删除的重复字段将为空。
有关什么是“重复”字段的信息，请后续的<a href="#syntax">协议缓冲区定义语法</a>小节。

新代码也将透明地读取旧消息。旧消息中不会出现新字段；
在这些情况下，协议缓冲区提供了一个合理的默认值。

### 什么时候 Protocol Buffers 不适合？ {#not-good-fit}

协议缓冲区并不适合所有数据。尤其：
- 协议缓冲区倾向于假设整个消息可以一次加载到内存中并且不大于对象图。
  对于超过几兆字节的数据，考虑不同的解决方案；
  在处理较大的数据时，由于序列化副本，你可能会有效地获得多个数据副本，
  这可能会导致内存使用量出现惊人的峰值。
- 当协议缓冲区被序列化时，相同的数据可以有许多不同的二进制序列化。
  如果不完全解析它们，就无法比较两条消息是否等同。
- 消息未压缩。虽然消息可以像任何其他文件一样被压缩或 gzip 压缩，
  但 JPEG 和 PNG 使用的专用压缩算法将为适当类型的数据生成更小的文件。
- 对于许多涉及大型多维浮点数数组的科学和工程用途，协议缓冲区消息在大小和速度方面都没有达到最大效率。
  对于这些应用程序，[FITS](https://en.wikipedia.org/wiki/FITS) 和类似格式的开销较小。
- 协议缓冲区在科学计算中流行的非面向对象语言（例如 Fortran 和 IDL）中没有得到很好的支持。
- 协议缓冲区消息本身并不自我描述其数据，但它们具有完全反射的模式，你可以使用它来实现自我描述。
  也就是说，如果不访问其相应的 `.proto` 文件，你将无法完全解释它。
- 协议缓冲区不是任何组织的正式标准。这使得它们不适合在具有法律或其他要求以建立在标准之上的环境中使用。

## 谁使用 Protocol Buffers？ {#who-uses}

许多外部可用的项目使用协议缓冲区，包括以下内容：
- [gRPC](https://grpc.io/)
- [Google Cloud](https://cloud.google.com/)
- [Envoy Proxy](https://www.envoyproxy.io/)

## Protocol Buffers 如何工作？ {#work}

下图显示了如何使用协议缓冲区来处理数据：

<center><img src="/posts/overview-of-protocal-buffers/workflow.png"/></center>

协议缓冲区生成的代码提供了实用方法来从文件和流中检索数据、从数据中提取单个值、检查数据是否存在、将数据序列化回文件或流以及其他有用的功能。

以下代码示例向你展示了此流程在 Java 中的示例。如前所示，这是一个 `.proto` 定义：

```protobuf
message Person {
  optional string name = 1;
  optional int32 id = 2;
  optional string email = 3;
}
```

编译此 `.proto` 文件会创建一个 Builder 类，你可以使用它来创建新实例，如以下 Java 代码所示：

```java
Person john = Person.newBuilder()
    .setId(1234)
    .setName("John Doe")
    .setEmail("jdoe@example.com")
    .build();
output = new FileOutputStream(args[0]);
john.writeTo(output);
```

然后，你可以使用协议缓冲区在其他语言（如 C++）中创建的方法反序列化数据：

```c++
Person john;
fstream input(argv[1], ios::in | ios::binary);
john.ParseFromIstream(&input);
int id = john.id();
std::string name = john.name();
std::string email = john.email();
```

## Protocol Buffers 定义语法 {#syntax}

定义 `.proto` 文件时，你可以指定字段是 `optional` 或 `repeated`（proto2 和 proto3）或 `singular`（proto3）。 
（proto3 中不存在将字段设置为 `required` 的情况，并且在 proto2 中强烈反对。
有关此的更多信息，请参阅
[指定字段规则](https://developers.google.com/protocol-buffers/docs/proto#specifying-rules)
中的 "Required is Forever"。）

在设置字段的可选性/可重复性后，你指定数据类型。
协议缓冲区支持通常的原始数据类型，例如整数、布尔值和浮点数。
有关完整列表，请参阅[标量值类型](https://developers.google.com/protocol-buffers/docs/proto#scalar)。

一个字段也可以是：
- `message` 类型，你可以嵌套部分定义，例如用于重复数据集。
- `enum` 类型，你可以指定一组值以供选择。
- `oneof` 类型，当消息有多个可选字段且最多同时设置一个字段时，可以使用该类型。
- `map` 类型，用于将键值定义。

在 proto2 中，消息可以允许扩展定义消息本身之外的字段。
例如，protobuf 库的内部消息模式允许扩展自定义的、特定于使用的选项。

有关可用选项的更多信息，请参阅
[proto2](https://developers.google.com/protocol-buffers/docs/proto)
或 [proto3](https://developers.google.com/protocol-buffers/docs/proto3) 的语言指南。

设置可选属性和字段类型后，你应该分配一个字段编号。
字段编号不能改变或重复。
如果你删除一个字段，你应该保留其字段编号，以防止有人意外重复该编号。

## 额外的数据类型支持 {#data-types}

协议缓冲区支持许多标量值类型，包括使用可变长度编码和固定大小的整数。
你还可以通过定义消息来创建自己的复合数据类型，这些消息本身就是可以分配给字段的数据类型。
除了简单和复合值类型之外，还发布了几种常见类型。

### 常见类型  {#common-types}

- [Duration](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/duration.proto)
  是有符号的、固定长度的时间跨度，例如 42s。
- [Timestamp](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/timestamp.proto)
  是独立于任何时区或日历的时间点，例如 2017-01-15T01:30:15.01Z。
- [Interval](https://github.com/googleapis/googleapis/blob/master/google/type/interval.proto)
  是独立于时区或日历的时间间隔，例如 2017-01-15T01:30:15.01Z - 2017-01-16T02:30:15.01Z。
- [Date](https://github.com/googleapis/googleapis/blob/master/google/type/date.proto)
  是一个完整的日历日期，例如 2025-09-19。
- [DayOfWeek](https://github.com/googleapis/googleapis/blob/master/google/type/dayofweek.proto)
  是一周中的某一天，例如 Monday。
- [TimeOfDay](https://github.com/googleapis/googleapis/blob/master/google/type/timeofday.proto)
  是一天中的某个时间，例如 10:42:23。
- [LatLng](https://github.com/googleapis/googleapis/blob/master/google/type/latlng.proto)
  是一个纬度/经度对，例如 37.386051 纬度和 -122.083855 经度。
- [Money](https://github.com/googleapis/googleapis/blob/master/google/type/money.proto)
  是具有货币类型的货币数量，例如 42 USD。
- [PostalAddress](https://github.com/googleapis/googleapis/blob/master/google/type/postal_address.proto)
  是一个邮政地址，例如 1600 Amphitheatre Parkway Mountain View, CA 94043 USA。
- [Color](https://github.com/googleapis/googleapis/blob/master/google/type/color.proto)
  是 RGBA 颜色空间中的一种颜色。
- [Month](https://github.com/googleapis/googleapis/blob/master/google/type/month.proto)
  是一年中的某个月，例如 April（四月）。

### Protocol Buffers 开源哲学 {#protocol_buffers_open_source_philosophy}

协议缓冲区于 2008 年开源，旨在为 Google 以外的开发人员提供与我们在内部从中获得的相同好处的一种方式。
我们通过定期更新语言来支持开源社区，因为我们进行这些更改以支持我们的内部需求。
虽然我们接受来自外部开发人员的精选拉取请求，但我们不能总是优先考虑不符合 Google 特定需求的功能请求和错误修复。