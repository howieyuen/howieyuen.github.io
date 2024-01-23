---
title: 语言指南（proto3）
date: 2022-05-10
categories: [Protobuf]
weight: 2
---

> 原文链接：[Language Guide(proto3) | Protocol Buffers | Google Developers](https://developers.google.com/protocol-buffers/docs/proto3)

<!-- 
This guide describes how to use the protocol buffer language to structure your protocol buffer data, including .proto file syntax and how to generate data access classes from your .proto files. It covers the proto3 version of the protocol buffers language: for information on the proto2 syntax, see the Proto2 Language Guide.

This is a reference guide – for a step by step example that uses many of the features described in this document, see the tutorial for your chosen language (currently proto2 only; more proto3 documentation is coming soon).
-->

本指南描述了如何使用 protobuf 语言结构化你的协议缓冲区数据，
包括 `.proto` 文件语法和如何从你的 `.proto` 文件生成数据访问的类（Class）。
它覆盖了 protobuf 语言的 **proto3** 版本：对于 **proto2** 语言的信息，
请查阅 [Proto2 Language Guide](https://developers.google.com/protocol-buffers/docs/proto)。

这是一篇参考指南——有关使用本文档中描述的许多功能的分步示例，
请参阅你选择的语言的
[教程](https://developers.google.com/protocol-buffers/docs/tutorials)（目前仅有 proto2；更多 proto3 文档即将推出）。

<!--more-->

## 定义消息类型 {#simple}

<!-- 
First let's look at a very simple example. Let's say you want to define a search request message format, where each search request has a query string, the particular page of results you are interested in, and a number of results per page. Here's the .proto file you use to define the message type.
-->

首先让我们看一个非常简单的例子。假设你要定义搜索请求消息格式，
其中每个搜索请求都有一个查询字符串、你感兴趣的特定结果页面以及每页的结果数量。
这是用于定义消息类型的 `.proto` 文件。

```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

<!-- 
- The first line of the file specifies that you're using proto3 syntax: if you don't do this the protocol buffer compiler will assume you are using proto2. This must be the first non-empty, non-comment line of the file.
- The SearchRequest message definition specifies three fields (name/value pairs), one for each piece of data that you want to include in this type of message. Each field has a name and a type.
-->
- 该文件的第一行指定你使用的是 proto3 语法：如果你不这样做，protocol buffer 编译器将假定你使用的是 proto2。
  这必须是文件的第一个非空、非注释行。
- `SearchRequest` 消息定义指定了三个字段（名称/值对），每个字段代表你希望此消息中包含的数据。
  每个字段都有一个名称和一个类型。

### 指定字段类型 {#specifying_field_types}

<!-- 
In the above example, all the fields are scalar types: two integers (page_number and result_per_page) and a string (query). However, you can also specify composite types for your fields, including enumerations and other message types.
-->
在上面的例子中，所有字段都是[标量类型](https://developers.google.com/protocol-buffers/docs/proto3#scalar)：
2 个整型（`page_number` 和 `result_per_page`）和 1 个字符串类型（`query`）。
然而，你也可以给你的字段指定复杂的类型，包括[枚举](https://developers.google.com/protocol-buffers/docs/proto3#enum)
和其他消息类型。

### 分配字段编号 {#assigning_field_numbers}

<!-- 
As you can see, each field in the message definition has a unique number. These field numbers are used to identify your fields in the message binary format, and should not be changed once your message type is in use. Note that field numbers in the range 1 through 15 take one byte to encode, including the field number and the field's type (you can find out more about this in Protocol Buffer Encoding). Field numbers in the range 16 through 2047 take two bytes. So you should reserve the numbers 1 through 15 for very frequently occurring message elements. Remember to leave some room for frequently occurring elements that might be added in the future.
-->
如你所见，消息定义中的每个字段都有一个**唯一编号**。
这些字段编号用于以 [消息二进制格式](https://developers.google.com/protocol-buffers/docs/encoding) 标识字段，编号一旦使用，你的消息类型就不应更改。
请注意，1 到 15 范围内的字段编号需要一个字节进行编码，
包括字段编号和字段类型（你可以在 [protobuf 编码](https://developers.google.com/protocol-buffers/docs/encoding#structure) 中找到更多相关信息）。
16 到 2047 范围内的字段编号占用两个字节。
因此，你应该为非常频繁出现的消息元素保留数字 1 到 15。
请记住为将来可能添加的频繁出现的元素留出一些空间。

<!-- 
The smallest field number you can specify is 1, and the largest is 229 - 1, or 536,870,911. You also cannot use the numbers 19000 through 19999 (FieldDescriptor::kFirstReservedNumber through FieldDescriptor::kLastReservedNumber), as they are reserved for the Protocol Buffers implementation - the protocol buffer compiler will complain if you use one of these reserved numbers in your .proto. Similarly, you cannot use any previously reserved field numbers.
-->
你可以指定的最小字段编号是 1，最大的是 2^29 - 1，即 536,870,911。你也不能使用数字 19000 到 19999
（`FieldDescriptor::kFirstReservedNumber` 到 `FieldDescriptor::kLastReservedNumber`），
因为它们是为 protobuf 实现保留的 - 如果你在 `.proto` 中使用这些保留数字之一，protobuf 编译器会报错。
同样，你不能使用任何以前[保留](https://developers.google.com/protocol-buffers/docs/proto3#reserved) 的字段编号。

### 指定字段规则 {#specifying_field_rules}
<!-- 
Message fields can be one of the following:

- singular: a well-formed message can have zero or one of this field (but not more than one). And this is the default field rule for proto3 syntax.
- repeated: this field can be repeated any number of times (including zero) in a well-formed message. The order of the repeated values will be preserved.
In proto3, repeated fields of scalar numeric types use packed encoding by default.

You can find out more about packed encoding in Protocol Buffer Encoding.
-->
Message 字段可以是以下其一：
- 单数：格式良好的消息可以有零个或一个此字段（但不能超过一个）。这是 proto3 语法的默认字段规则。
- 复数（`repeated`）：此字段可以在格式良好的消息中重复任意次数（包括零次）。重复值的顺序将被保留。
  在 proto3 中，标量数字类型的重复字段默认使用打包编码。

你可以在 [Protocol Buffer Encoding](https://developers.google.com/protocol-buffers/docs/encoding#packed) 中找到有关打包编码的更多信息。

### 添加更多的消息类型 {#adding_more_message_types}

<!-- 
Multiple message types can be defined in a single .proto file. This is useful if you are defining multiple related messages – so, for example, if you wanted to define the reply message format that corresponds to your SearchResponse message type, you could add it to the same .proto:
-->
可以在单个 .proto 文件中定义多种消息类型。如果你要定义多个相关消息，这很有用——例如，如果你你定义与你的 SearchResponse 消息类型相对应的回复消息格式，你可以将其添加到同一个 .proto 中：

```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

### 添加注释 {#adding_comments}
<!-- 
To add comments to your .proto files, use C/C++-style // and /* ... */ syntax.
-->
要向 `.proto` 文件中添加注释，请使用 C/C++ 风格的 `//` 和 `/* ... */` 语法：

```protobuf
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
``` 

### 保留字段 {#reserved}

<!-- 
If you update a message type by entirely removing a field, or commenting it out, future users can reuse the field number when making their own updates to the type. This can cause severe issues if they later load old versions of the same .proto, including data corruption, privacy bugs, and so on. One way to make sure this doesn't happen is to specify that the field numbers (and/or names, which can also cause issues for JSON serialization) of your deleted fields are reserved. The protocol buffer compiler will complain if any future users try to use these field identifiers.
-->
如果你通过完全删除某个字段或将其注释掉来<a href="#updating">更新</a>消息类型，未来的用户可以在对类型进行自己的更新时重新使用该字段编号。
如果他们稍后加载相同 `.proto` 的旧版本，这可能会导致严重的问题，包括数据损坏、隐私错误等。
确保不会发生这种情况的一种方法是指定保留已删除字段的字段编号（和/或名称，这也可能导致 JSON 序列化问题）。
如果将来有任何用户尝试使用这些字段标识符，protobuf 编译器会报错。

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
<!-- 
Note that you can't mix field names and field numbers in the same reserved statement.
-->
请注意，你不能在同一保留语句里，混合字段名和字段编号。

### 你的 .proto 文件生成了什么？ {#whats_generated_from_your_proto}

当你在 .proto 上运行 <a href="#generating"> protobuf 编译器</a>时，编译器会以你选择的语言生成代码，
你需要使用文件中描述的消息类型，包括获取和设置字段值，将消息序列化为输出流，并从输入流中解析你的消息。

- 对于 `C++`，编译器从每个 `.proto` 生成一个 `.h` 和 `.cc` 文件，并为文件中描述的每种消息类型提供一个类。
- 对于 `Java`，编译器会生成一个 `.java` 文件，其中包含每个消息类型的类，以及用于创建消息类实例的特殊 `Builder` 类。
- 对于 `Kotlin`，除了 `Java` 生成的代码之外，编译器还会为每种消息类型生成一个 `.kt` 文件，其中包含可用于简化创建消息实例的 DSL。
- Python 有点不同——Python 编译器会生成一个模块，其中包含 `.proto` 中每种消息类型的静态描述符，然后将其与元类一起用于在运行时创建必要的 Python 数据访问类。
- 对于 `Go`，编译器会生成一个 `.pb.go` 文件，其中包含文件中每种消息类型的类型。
- 对于 `Ruby`，编译器会生成一个 `.rb` 文件，其中包含一个包含消息类型的 Ruby 模块。
- 对于 `Objective-C`，编译器从每个 `.proto` 生成一个 `pbobjc.h` 和 `pbobjc.m` 文件，并为文件中描述的每种消息类型提供一个类。
- 对于 `C#`，编译器从每个 `.proto` 生成一个 `.cs` 文件，其中包含文件中描述的每种消息类型的类。
- 对于 `Dart`，编译器会生成一个 `.pb.dart` 文件，其中包含文件中每种消息类型的类。

你可以按照所选语言的教程（即将推出 proto3 版本）了解有关使用每种语言的 API 的更多信息。
有关更多 API 详细信息，请参阅相关 [API 参考](https://developers.google.com/protocol-buffers/docs/reference/overview)（proto3 版本也即将推出）。

## 标量值类型 {#scalar}
<!--
A scalar message field can have one of the following types – the table shows the type specified in the .proto file, and the corresponding type in the automatically generated class:
-->
标量消息字段可以具有以下类型之一。该表显示了 .proto 文件中指定的类型，以及自动生成的类中的相应类型：

| .proto | C++ | Java/Kotlin[1] | Python[3] | Go | Ruby | C# | PHP |  Dart |
| ----   | --- | -------------- | --------- | -- | ---- | -- | --- | ----- |
| double | double | double | float | float64 | Float | double | float | double |
| float | float | float | float | float32 | Float | float | float | double |
| int32 | int32 | int | int | int32 |Fixnum or Bignum (as required) | int | integer | int |
| int64 | int64 | long | int/long[4] | int64 | Bignum | long | integer/string[6] | Int64 |
| uint32 | uint32 | int[2] | int/long[4] | uint32 | Fixnum or Bignum (as required) | uint | integer | int |
| uint64 | uint64 | long[2] | int/long[4] | uint64 | Bignum | ulong | integer/string[6] | Int64 | 
| sint32 | int32 | int | int | int32 | Fixnum or Bignum (as required) | int | integer | int |
| sint64 | int64 | long | int/long[4] | int64 | Bignum long | integer/string[6]| Int64 |
| fixed32 | uint32 | int[2] | int/long[4] | uint32 | Fixnum or Bignum (as required) | uint | integer | int |
| fixed64 | uint64 | long[2] | int/long[4] | uint64 | Bignum | ulong | integer/string[6] | Int64 |
| sfixed32 | int32 | int | int | int32 | Fixnum or Bignum (as required) | int | integer | int |
| sfixed64 | int64 | long | int/long[4] | int64 | Bignum | long | integer/string[6] | Int64 |
| bool | bool | boolean | bool | bool | TrueClass/FalseClass | bool | boolean | bool |
| string | string | String | str/unicode[5] | string | String (UTF-8) | string | string | String |
| bytes | string | ByteString | str (Python 2) | bytes (Python 3) | []byte | String (ASCII-8BIT) | ByteString | string | List |

{{< hint info >}}
注意：
- int32：使用可变长度编码。对负数进行编码效率低下。如果你的字段可能有负值，请改用 sint32。
- int64：使用可变长度编码。对负数进行编码效率低下。如果你的字段可能有负值，请改用 sint64。
- uint32：使用可变长度编码。
- uint64：使用可变长度编码。
- sint32：使用可变长度编码。带符号的 int 值。这些比常规 int32 更有效地编码负数。
- sint64：使用可变长度编码。带符号的 int 值。这些比常规 int64 更有效地编码负数。
- fixed32：总是四个字节。如果值通常大于 2^28，则比 uint32 更有效。
- fixed64：总是八个字节。如果值通常大于 2^56，则比 uint64 更有效。
- sfixed32：总是四个字节。
- sfixed64：总是八个字节。
- string：字符串必须始终包含 UTF-8 编码或 7 位 ASCII 文本，并且不能超过 2^32。
- bytes：可以包含不超过 2^32 的任意字节序列。
{{< /hint >}}

<!-- 
You can find out more about how these types are encoded when you serialize your message in Protocol Buffer Encoding.

[1] Kotlin uses the corresponding types from Java, even for unsigned types, to ensure compatibility in mixed Java/Kotlin codebases.

[2] In Java, unsigned 32-bit and 64-bit integers are represented using their signed counterparts, with the top bit simply being stored in the sign bit.

[3] In all cases, setting values to a field will perform type checking to make sure it is valid.

[4] 64-bit or unsigned 32-bit integers are always represented as long when decoded, but can be an int if an int is given when setting the field. In all cases, the value must fit in the type represented when set. See [2].

[5] Python strings are represented as unicode on decode but can be str if an ASCII string is given (this is subject to change).

[6] Integer is used on 64-bit machines and string is used on 32-bit machines.
-->
当你在 [protobuf 编码](https://developers.google.com/protocol-buffers/docs/encoding) 中序列化你的消息时，你可以了解有关这些类型如何编码的更多信息。

[1] Kotlin 使用 Java 中的相应类型，甚至是无符号类型，以确保在混合 Java/Kotlin 代码库中的兼容性。

[2] 在 Java 中，无符号 32 位和 64 位整数使用它们的有符号对应物表示，最高位简单地存储在符号位中。

[3] 在所有情况下，为字段设置值将执行类型检查以确保其有效。

[4] 64 位或无符号 32 位整数在解码时始终表示为 long，但如果在设置字段时给出 int，则可以是 int。在所有情况下，该值必须适合设置时表示的类型。见[2]。

[5] Python 字符串在解码时表示为 unicode，但如果给出 ASCII 字符串，则可以是 str（这可能会发生变化）。

[6] 整数用于 64 位机器，字符串用于 32 位机器。

## 默认值 {#default}
<!-- 
When a message is parsed, if the encoded message does not contain a particular singular element, the corresponding field in the parsed object is set to the default value for that field. These defaults are type-specific:
-->
解析消息时，如果编码的消息不包含特定的单个元素，则解析对象中的相应字段将设置为该字段的默认值。这些默认值是特定于类型的：

<!-- 
For strings, the default value is the empty string.
For bytes, the default value is empty bytes.
For bools, the default value is false.
For numeric types, the default value is zero.
For enums, the default value is the first defined enum value, which must be 0.
For message fields, the field is not set. Its exact value is language-dependent. See the generated code guide for details.
-->
- 对于字符串，默认值为空字符串。
- 对于字节，默认值为空字节。
- 对于布尔值，默认值为 false。
- 对于数字类型，默认值为零。
- 对于<a href="#enum">枚举</a>，默认值是**首个定义的枚举值**，必须为 0。
- 对于消息字段，未设置该字段。它的确切值取决于语言。有关详细信息，请参阅<a href="overview-of-protocal-buffers.md">生成代码指南</a>。

<!-- 
The default value for repeated fields is empty (generally an empty list in the appropriate language).

Note that for scalar message fields, once a message is parsed there's no way of telling whether a field was explicitly set to the default value (for example whether a boolean was set to false) or just not set at all: you should bear this in mind when defining your message types. For example, don't have a boolean that switches on some behavior when set to false if you don't want that behavior to also happen by default. Also note that if a scalar message field is set to its default, the value will not be serialized on the wire.

See the generated code guide for your chosen language for more details about how defaults work in generated code.
-->
重复字段的默认值为空（通常是相应语言的空列表）。

请注意，对于标量消息字段，一旦解析了消息，就无法判断一个字段是显式设置为默认值（例如布尔值是否设置为 `false`）还是根本没有设置：你应该在定义消息类型时请注意。
例如，如果你不希望在默认情况下也发生该行为，则不要在设置为 false 时打开某些行为的布尔值。
另请注意，如果标量消息字段设置为其默认值，则该值将不会在线上序列化。

有关默认值如何在生成的代码中工作的更多详细信息，请参阅你选择的语言的<a href="overview-of-protocal-buffers.md">生成代码指南</a>。

## 枚举 {#enum}

在定义消息类型时，你可能希望其字段之一仅具有预定义的值列表之一。
例如，假设你要为每个 SearchRequest 添加一个 `corpus` 字段，其中 `corpus` 可以是 `UNIVERSAL`、`WEB`、`IMAGES`、`LOCAL`、`NEWS`、`PRODUCTS` 或 `VIDEO`。
你可以通过在消息定义中添加一个 `enum`，即可非常简单地做到这一点，每个可能的值都有一个常量。

在以下示例中，我们添加了一个名为 `Corpus` 的 `enum`，其中包含所有可能的值，以及一个 `Corpus` 类型的字段：

```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```

如你所见，`Corpus` 枚举的第一个常量映射到零：每个枚举定义**必须**包含一个映射到零的常量作为其第一个元素。这是因为：

- 必须有一个零值，以便我们可以使用 0 作为数字<a href="#default">默认值</a>。
- 零值必须是第一个元素，以便与第一个枚举值始终为默认值的 [proto2](https://developers.google.com/protocol-buffers/docs/proto) 语义兼容。

你可以通过将相同的值分配给不同的枚举常量来定义别名。为此，你需要将 `allow_alias` 选项设置为 `true`，否则协议编译器将在找到别名时生成错误消息。

```protobuf
message MyMessage1 {
  enum EnumAllowingAlias {
    option allow_alias = true;
    UNKNOWN = 0;
    STARTED = 1;
    RUNNING = 1;
  }
}
message MyMessage2 {
  enum EnumNotAllowingAlias {
    UNKNOWN = 0;
    STARTED = 1;
    // RUNNING = 1;  // 取消注释此行将导致 Google 内部出现编译错误，外部出现警告消息。
  }
}
```

枚举器常量必须在 32 位整数范围内。由于 `enum` 在线路上使用 [varint 编码](https://developers.google.com/protocol-buffers/docs/encoding)，
因此负数效率低下，因此不推荐使用。你可以在消息定义中定义 `enum`，如上例所示，也可以在外部定义 `enum`，这些 `enum` 可以在 `.proto` 文件中的任何消息定义中复用。
你还可以使用在一条消息中声明的 `enum` 类型作为另一条消息中字段的类型，使用语法 `_MessageType_._EnumType_`。

当你在使用 `enum` 的 .proto 上运行协议缓冲区编译器时，生成的代码将具有对应于 Java、Kotlin 或 C++ 的 `enum`，
或用于 Python 的特殊 `EnumDescriptor` 类，用于创建一组带整数的符号常量运行时生成的类中的值。

{{< hint danger >}}
注意：生成的代码可能会受到特定于语言的枚举数限制（一种语言的低至数千）。请查看你计划使用的语言的限制。
{{< /hint >}}

在反序列化期间，无法识别的枚举值将保留在消息中，尽管在反序列化消息时如何表示这取决于语言。
在支持具有超出指定符号范围的值的开放枚举类型的语言中，例如 C++ 和 Go，未知的枚举值简单地存储为其底层整数表示。
在 Java 等具有封闭枚举类型的语言中，枚举中的 case 用于表示无法识别的值，并且可以使用特殊的访问器访问底层整数。
在任何一种情况下，如果消息被序列化，则无法识别的值仍将与消息一起序列化。

有关如何在应用程序中使用消息 `enum` 的更多信息，请参阅所选语言的<a href="overview-of-protocal-buffers.md">生成代码指南</a>。

### 保留值 {#reserved_values}

如果你通过完全删除枚举条目或将其注释掉来<a href="#updating">更新</a>枚举类型，将来的用户可以在对类型进行自己的更新时重用该数值。
如果他们稍后加载相同 `.proto` 的旧版本，这可能会导致严重的问题，包括数据损坏、隐私错误等。
确保不会发生这种情况的一种方法是指定保留已删除条目的数值（和/或名称，这也可能导致 JSON 序列化问题）。
如果将来有任何用户尝试使用这些标识符，protocol buffer 编译器会抱怨。你可以使用 `max` 关键字指定保留的数值范围达到最大可能值。

```protobuf
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

请注意，你不能在同一保留语句中混合字段名称和数值。

## 使用其他消息类型 {#other}

你可以使用其他消息类型作为字段类型。例如，假设你想在每个 `SearchResponse` 消息中包含 `Result` 消息。
为此，你可以在同一个 `.proto` 中定义一个 Result 消息类型，然后在 `SearchResponse` 中指定一个 `Result` 类型的字段：

```protobuf
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

### 引入定义 {#importing_definitions}

默认情况下，你只能使用直接导入的 `.proto` 文件中的定义。但是，有时你可能需要将 `.proto` 文件移动到新位置。
你可以在旧位置放置一个占位符 `.proto` 文件，以使用 `import public` 概念将所有导入转发到新位置，而不是直接移动 `.proto` 文件并在一次更改中更新所有调用站点。

**请注意，公共导入功能在 Java 中不可用。**

任何导入包含 `import public` 语句的 proto 的代码都可以传递依赖 `import public` 依赖项。例如：

```protobuf
// new.proto
// 所有定义被移动到这里
```

```protobuf
// old.proto
// 这是所有客户端正在导入的原型。
import public "new.proto";
import "other.proto";
```

```protobuf
// client.proto
import "old.proto";
// 你使用 old.proto 和 new.proto 中的定义，而不是 other.proto
```

协议编译器使用 `-I/--proto_path` 标志在协议编译器命令行上指定的一组目录中搜索导入的文件。
如果没有给出标志，它会在调用编译器的目录中查找。通常，你应该将 `--proto_path` 标志设置为项目的根目录，并为所有导入使用完全限定名称。

### 使用 proto2 消息类型 {#using_proto2_message_types}

可以导入 `proto2` 消息类型并在你的 proto3 消息中使用它们，反之亦然。
但是，proto2 枚举不能直接在 proto3 语法中使用（如果导入的 proto2 消息使用它们也没关系）。

## 内嵌类型 {#nested}

你可以在其他消息类型中定义和使用消息类型，如下例所示，这里 `Result` 消息在 `SearchResponse` 消息中定义：

```protobuf
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

如果你想在其父消息类型之外重用此消息类型，则将其称为 `_Parent_._Type_`：

```protobuf
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```

你可以随意嵌套消息：

```protobuf
message Outer {                  // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      int64 ival = 1;
      bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      int32 ival = 1;
      bool  booly = 2;
    }
  }
}
```

## 更新消息类型 {#updating}

如果现有的消息类型不再满足你的所有需求。例如，你希望消息格式有一个额外的字段，但你仍然希望使用使用旧格式创建的代码，请不要担心！
在不破坏任何现有代码的情况下更新消息类型非常简单。只需记住以下规则：

- 不要更改任何现有字段的字段编号。
- 如果你添加新字段，则使用“旧”消息格式的代码序列化的任何消息仍然可以由新生成的代码解析。
  你应该记住这些元素的<a href="#default">默认值</a>，以便新代码可以正确地与旧代码生成的消息交互。
  类似地，新代码创建的消息可以由旧代码解析：旧二进制文件在解析时会忽略新字段。有关详细信息，请参阅<a href="unknowns">未知字段</a>部分。
- 只要在更新的消息类型中不再使用字段编号，就可以删除字段。
  你可能想要重命名该字段，可能添加前缀“OBSOLETE_”，或保留字段编号，以便你的 `.proto` 的未来用户不会意外重用该编号。
- `int32`、`uint32`、`int64`、`uint64` 和 `bool` 都是兼容的——这意味着你可以将字段从其中一种类型更改为另一种类型，而不会破坏前向或后向兼容性。
   如果从不适合相应类型的线路中解析出一个数字，你将获得与在 C++ 中将该数字强制转换为该类型相同的效果（例如，如果一个 64 位数字被读取为 int32，它将被截断为 32 位）。
- `sint32` 和 `sint64` 相互兼容，但与其他整数类型不兼容。
- 只要字节是有效的 UTF-8，`string` 和 `byte` 就兼容。
- 如果字节包含消息的编码版本，则嵌入消息与 `byte` 兼容。
- `fixed32` 与 `sfixed32` 兼容，`fixed64` 与 `sfixed64` 兼容。
- 对于 `string`、`byte` 和 消息字段，`optional` 与 `repeated` 兼容。给定一个重复字段的序列化数据作为输入，
  如果它是一个原始类型字段，那么期望这个字段是 `optional` 的客户端将采用最后一个输入值，
  或者如果它是一个消息类型字段，则合并所有输入元素。
  请注意，这对于数字类型（包括布尔值和枚举）通常不安全。
  数字类型的重复字段可以以<a href="https://developers.google.com/protocol-buffers/docs/encoding#packed">打包</a>格式序列化，当需要 `optional` 字段时，将无法正确解析。
- `enum` 在有线格式方面与 `int32`、`uint32`、`int64` 和 `uint64` 兼容（请注意，如果值不合适，将被截断）。
  但是请注意，当消息被反序列化时，客户端代码可能会以不同的方式对待它们：
  例如，无法识别的 proto3 `enum` 类型将保留在消息中，但是当消息被反序列化时如何表示则取决于语言。Int 字段总是只保留它们的值。
- 将单个值更改为新 `oneof` 的成员是安全且二进制兼容的。
  如果你确定没有代码一次设置多个字段，则将多个字段移动到新的 `oneof` 中可能是安全的。
  将任何字段移动到现有的 `oneof` 中是不安全的。

## 未知字段 {#unknowns}

未知字段是格式良好的协议缓冲区序列化数据，表示解析器无法识别的字段。
例如，当旧二进制文件用新字段解析新二进制文件发送的数据时，这些新字段将成为旧二进制文件中的未知字段。

最初，proto3 消息在解析过程中总是丢弃未知字段，但在 3.5 版本中，我们重新引入了保留未知字段以匹配 proto2 行为。
在 3.5 及更高版本中，未知字段在解析期间保留并包含在序列化输出中。

## Any {#any}

`Any` 消息类型允许你将消息用作嵌入类型，而无需定义它们的 `.proto`。
`Any` 包含作为 `byte` 的任意序列化消息，以及充当全局唯一标识符并解析为该消息类型的 URL。
要使用 `Any` 类型，你需要<a href="#other">导入</a> `google/protobuf/any.proto`。

```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

给定消息类型的默认类型 URL 是 `type.googleapis.com/_packagename_._messagename_`。

不同的语言实现将支持运行时库助手以类型安全的方式打包和解包 `Any` 值。
例如，在 Java 中，`Any` 类型将具有特殊的 `pack()` 和 `unpack()` 访问器，而在 C++ 中则有 `PackFrom()` 和 `UnpackTo()` 方法：

```protobuf
// 在 Any 中存储任意类型消息
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// 从 Any 中读取任意消息
ErrorStatus status = ...;
for (const Any& detail : status.details()) {
  if (detail.Is<NetworkErrorDetails>()) {
    NetworkErrorDetails network_error;
    detail.UnpackTo(&network_error);
    ... processing network_error ...
  }
}
```

**目前，用于处理 `Any` 类型的运行时库正在开发中。**

如果你已经熟悉 [proto2 语法](https://developers.google.com/protocol-buffers/docs/proto)，
Any 可以保存任意 proto3 消息，类似于可以允许<a href="https://developers.google.com/protocol-buffers/docs/proto#extensions">扩展</a>的 proto2 消息。

## Oneof {#oneof}

如果你有一条包含多个字段的消息，并且最多同时设置一个字段，你可以强制执行此行为并使用 `oneof` 功能节省内存。

`oneof` 字段与常规字段一样，除了一个 `oneof` 共享内存中的所有字段外，最多可以同时设置一个字段。
设置 `oneof` 的任何成员会自动清除所有其他成员。你可以使用特殊的 `case()` 或 `WhichOneof()` 方法检查 `oneof` 中设置的值（如果有），具体取决于你选择的语言。

### 使用 oneof {#using_oneof}

要在 `.proto` 中定义 `oneof`，请使用 `oneof` 关键字，后跟 `oneof` 名称，在本例中为 `test_oneof`：

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

然后，你将 `oneof` 字段添加到 `oneof` 定义中。你可以添加任何类型的字段，`map` 字段和 `repeated` 字段除外。

在你生成的代码中，`oneof` 字段具有与常规字段相同的 `getter` 和 `setter`。
你还可以获得一种特殊的方法来检查 `oneof` 中设置了哪个值（如果有）。
你可以在相关 <a href="overview-of-protocal-buffers.md">API 参考</a>中找到有关所选语言的 `oneof` API 的更多信息。

### oneof 功能 {#oneof_features}

- 设置 oneof 字段将自动清除 oneof 的所有其他成员。 因此，如果你设置了多个 oneof 字段，则只有你设置的最后一个字段仍有值。
  ```protobuf
  SampleMessage message;
  message.set_name("name");
  CHECK(message.has_name());
  message.mutable_sub_message();   // 将会清除字段名
  CHECK(!message.has_name());
  ```
- 如果解析器在线路上遇到同一个成员的多个成员，则在解析的消息中只使用最后一个看到的成员。
- oneof 字段不能是 `repeated`。
- 反射 API 适用于 oneof 字段。
- 如果将 oneof 字段设置为默认值（例如将 int32 oneof 字段设置为 0），则会设置该 oneof 字段的“大小写”，并且该值将在线上序列化。
- 如果你使用 C++，请确保你的代码不会导致内存崩溃。 以下示例代码将崩溃，因为 `sub_message` 已通过调用 `set_name()` 方法删除。
  ```protobuf
  SampleMessage message;
  SubMessage* sub_message = message.mutable_sub_message();
  message.set_name("name");      // 将会删除 sub_message
  sub_message->set_...           // 在此处崩溃
  ```
- 同样在 C++ 中，如果你 `Swap()` 两个 oneof 消息，则每条消息都会以另一个的 oneof 情况结束。
  在下面的示例中，`msg1` 将有一个 `sub_message`，而 `msg2` 将有一个名称。
  ```protobuf
  SampleMessage msg1;
  msg1.set_name("name");
  SampleMessage msg2;
  msg2.mutable_sub_message();
  msg1.swap(&msg2);
  CHECK(msg1.has_sub_message());
  CHECK(msg2.has_name());
  ```

### 向后兼容性问题 {#backwards-compatibility_issues}
添加或删除其中一个字段时要小心。如果检查 oneof 的值返回 `None/NOT_SET`，则可能意味着 oneof 尚未设置或已设置为 oneof 不同版本中的字段。
没有办法区分，因为无法知道线路上的未知字段是否是 oneof 的成员。

**标签重用问题**
- **将字段移入或移出 oneof**：在消息被序列化和解析后，你可能会丢失一些信息（某些字段将被清除）。
  但是，你可以安全地将单个字段移动到新的 oneof 中，并且如果知道只设置了一个字段，则可以移动多个字段。
- **删除 oneof 字段并重新添加**：这可能会在消息被序列化和解析后清除你当前设置的 oneof 字段。
- **拆分或合并其中一个**：这与移动常规字段有类似的问题。

## Map {#maps}

如果你想创建关联映射作为数据定义的一部分，protocol buffers 提供了一种方便的快捷语法：

```protobuf
map<key_type, value_type> map_field = N;
```

...其中 `key_type` 可以是任何整数或字符串类型（因此，除浮点类型和字节之外的任何标量类型）。
请注意，枚举不是有效的 `key_type`。`value_type` 可以是除另一个映射之外的任何类型。

因此，例如，如果你想创建一个项目映射，其中每个项目消息都与一个字符串键相关联，你可以这样定义它：

```protobuf
map<string, Project> projects = 3;
```

- map 字段不能 `repeated`。
- map 值的线格式排序和 map 迭代排序是未定义的，因此你不能依赖 map 项处于特定顺序。
- 为 `.proto` 生成文本格式时，map 按键排序。 数字键按数字排序。
- 从连线解析或合并时，如果有重复的映射键，则使用最后看到的键。从文本格式解析 map 时，如果有重复的键，则解析可能会失败。
- 如果你为映射字段提供键但没有值，则该字段被序列化时的行为取决于语言。在 C++、Java、Kotlin 和 Python 中，类型的默认值是序列化的，而在其他语言中则没有序列化。

生成的地图 API 目前可用于所有 proto3 支持的语言。你可以在相关 <a href="overview-of-protocal-buffers.md">API 参考</a>中找到有关所选语言的 map API 的更多信息。

### 向后兼容 {#backwards_compatibility}

map 语法在网络上等同于以下内容，因此不支持 map 的协议缓冲区实现仍然可以处理你的数据：

```protobuf
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

任何支持映射的协议缓冲区实现都必须生成和接受上述定义可以接受的数据。

## 包 {#packages}

你可以将可选的 `package` 说明符添加到 .proto 文件中，以防止协议消息类型之间的名称冲突。

```protobuf
package foo.bar;
message Open { ... }
```

然后，你可以在定义消息类型的字段时使用包说明符：

```protobuf
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```

包说明符影响生成代码的方式取决于你选择的语言：

- 在 **C++** 中，生成的类被包装在 C++ 命名空间中。例如，`Open` 将位于命名空间 `foo::bar` 中。
- 在 **Java** 和 **Kotlin** 中，包用作 Java 包，除非你在 `.proto` 文件中明确提供选项 `java_package` 。
- 在 **Python** 中，`package` 指令被忽略，因为 Python 模块是根据它们在文件系统中的位置来组织的。
- 在 **Go** 中，包用作 Go 包名称，除非你在 `.proto` 文件中明确提供选项 `go_package`。
- 在 **Ruby** 中，生成的类封装在嵌套的 Ruby 命名空间中，转换为所需的 Ruby 大写样式（第一个字母大写；如果第一个字符不是字母，则 `PB_` 被前置）。
  例如，`Open` 将位于命名空间 `Foo::Bar` 中。
- 在 **C#** 中，包在转换为 `PascalCase` 后用作命名空间，除非你在 `.proto` 文件中明确提供选项 `csharp_namespace`。
  例如，Open 将在命名空间 `Foo.Bar` 中。

### 包和名称解析 {#packages_and_name_resolution}

protobuf 中的类型名称解析与 C++ 类似：首先搜索最内部的范围，然后搜索下一个最内部的范围，依此类推，每个包都被认为是其父包的“内部”。
一个前置的 '.' （例如，`.foo.bar.Baz`）表示从最外面的范围开始。

协议缓冲区编译器通过解析导入的 `.proto` 文件来解析所有类型名称。
每种语言的代码生成器都知道如何引用该语言中的每种类型，即使它有不同的范围规则。

## 定义服务 {#services}

如果你想在 RPC（远程过程调用）系统中使用你的消息类型，你可以在 `.proto` 文件中定义一个 RPC 服务接口，并且协议缓冲区编译器将以你选择的语言生成服务接口代码和存根。
因此，例如，如果你想使用获取 `SearchRequest` 并返回 `SearchResponse` 的方法定义 RPC 服务，你可以在 .proto 文件中定义它，如下所示：

```protobuf
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

与协议缓冲区一起使用的最直接的 RPC 系统是 [gRPC](https://grpc.io/)：由 Google 开发的一种语言和平台中立的开源 RPC 系统。
gRPC 特别适用于协议缓冲区，并允许你使用特殊的协议缓冲区编译器插件直接从 `.proto` 文件生成相关的 RPC 代码。

如果你不想使用 gRPC，也可以将协议缓冲区与你自己的 RPC 实现一起使用。
你可以在 <a href="https://developers.google.com/protocol-buffers/docs/proto#services">Proto2 语言指南</a>中找到更多相关信息。

还有一些正在进行的第三方项目为 Protocol Buffers 开发 RPC 实现。
有关我们了解的项目的链接列表，请参阅<a href="https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md">第三方附加组件 wiki 页面</a>。

## JSON 映射 {#json}

Proto3 支持 JSON 中的规范编码，从而更容易在系统之间共享数据。下表中按类型描述了编码。

如果 JSON 编码的数据中缺少某个值，或者它的值为 `null`，则在解析到协议缓冲区时，它将被解释为适当的<a href="#default">默认值</a>。
如果某个字段在协议缓冲区中具有默认值，则在 JSON 编码的数据中默认将其省略以节省空间。
实现可以提供选项以在 JSON 编码的输出中发出具有默认值的字段。

| proto3 | JSON | JSON example |
| ------ | ---- | ------------ |
| message | object | {"fooBar": v, "g": null, …} |
| enum | string | "FOO_BAR" |
| map<K,V> |  object |  {"k": v, …} |
| repeated V |  array |  [v, …] |
| bool |  true, false |  true, false |  
| string |  string |  "Hello World!" |  
| bytes |  base64 string |  "YWJjMTIzIT8kKiYoKSctPUB+" |
| int32, fixed32, uint32 |  number |  1, -10, 0 |
| int64, fixed64, uint64 |  string |  "1", "-10" |
| float, double |  number |  1.1, -10.0, 0, "NaN", "Infinity" |
| Any |  object |  {"@type": "url", "f": v, … } |
| Timestamp |  string |  "1972-01-01T10:00:20.021Z" |
| Duration |  string |  "1.000340012s", "1s" |
| Struct |  object |  { … } |
| Wrapper types |  various types |  2, "2", "foo", true, "true", null, 0, … |
| FieldMask |  string |  "f.fooBar,h" |
| ListValue |  array |  [foo, bar, …] |  
| Value |  value |   |
| NullValue |  null |   |
| Empty |  object |  {} |

### JSON 选项 {#json_options}

proto3 JSON 实现可以提供以下选项：

- **发送具有默认值的字段**：在 proto3 JSON 输出中默认省略具有默认值的字段。实现可以提供一个选项来覆盖此行为并使用其默认值输出字段。
- **忽略未知字段**：Proto3 JSON 解析器默认应拒绝未知字段，但可能会提供在解析中忽略未知字段的选项。
- **使用 proto 字段名称而不是 lowerCamelCase 名称**：默认情况下，proto3 JSON 打印器应将字段名称转换为首字符小写的驼峰式并将其用作 JSON 名称。
  实现可能会提供一个选项来使用 proto 字段名称作为 JSON 名称。proto3 JSON 解析器需要接受转换后的首字符小写的驼峰式名称和 proto 字段名称。
- **将枚举值作为整数而不是字符串发送**：默认情况下，在 JSON 输出中使用枚举值的名称。可以提供一个选项来代替使用枚举值的数值。

## 选项 {#options}

`.proto` 文件中的单个声明可以使用多个选项进行注释。选项不会改变声明的整体含义，但可能会影响它在特定上下文中的处理方式。
可用选项的完整列表在 `google/protobuf/descriptor.proto` 中定义。

一些选项是文件级选项，这意味着它们应该在顶级范围内编写，而不是在任何消息、枚举或服务定义中。
一些选项是消息级别的选项，这意味着它们应该写在消息定义中。
一些选项是字段级选项，这意味着它们应该写在字段定义中。
选项也可以写在枚举类型、枚举值、oneof 字段、服务类型和服务方法上；但是，目前不存在任何有用的选项。

以下是一些最常用的选项：
- `java_package`（文件选项）：要用于生成的 Java/Kotlin 类的包。
  如果 `.proto` 文件中没有明确给出 `java_package` 选项，则默认使用 proto 包（使用 `.proto` 文件中的 “package” 关键字指定）。
  但是，proto 包通常不能制作好的 Java 包，因为不期望 proto 包以反向域名开头。如果不生成 Java 或 Kotlin 代码，则此选项无效。
  ```protobuf
  option java_package = "com.example.foo";
  ```
- `java_outer_classname`（文件选项）：要生成的包装 Java 类的类名（以及文件名）。
  如果 .proto 文件中没有明确指定 `java_outer_classname`，则将通过将 `.proto` 文件名转换为驼峰式来构造类名（因此 `foo_bar.proto` 变为 `FooBar.java`）。
  如果 `java_multiple_files` 选项被禁用，那么所有其他 class/enum 等为 `.proto` 文件生成的文件将在这个外部包装 Java 类中生成为嵌套 class/enum 等。
  如果不生成 Java 代码，则此选项无效。

  ```protobuf
  option java_outer_classname = "Ponycopter";
  ```

- `java_multiple_files`（文件选项）：如果为 false，则只会为此 .proto 文件以及所有 Java class/enum 等生成一个 `.java` 文件。
  为顶级消息、服务和枚举生成的将嵌套在外部类中（请参阅 `java_outer_classname`）。 如果为 true，将为每个 Java class/enum 等生成单独的 `.java` 文件。
  为顶级消息、服务和枚举生成，并且为此 `.proto` 文件生成的包装 Java 类将不包含任何嵌套 class/enum 等。
  这是一个布尔选项，默认为 `false`。如果不生成 Java 代码，则此选项无效。
  ```protobuf
  option java_multiple_files = true;
  ```

- `optimize_for`（文件选项）：可以设置为 `SPEED`、`CODE_SIZE` 或 `LITE_RUNTIME`。这会通过以下方式影响 C++ 和 Java 代码生成器（可能还有第三方生成器）：
  - `SPEED`（默认）：protocol buffer 编译器将生成用于序列化、解析和对消息类型执行其他常见操作的代码。此代码经过高度优化。
  - `CODE_SIZE`：protocol buffer 编译器将生成最少的类，并将依赖共享的、基于反射的代码来实现序列化、解析和各种其他操作。因此，生成的代码将比使用 `SPEED` 小得多，但操作会更慢。
    类仍将实现与在 SPEED 模式下完全相同的公共 API。此模式在包含大量 `.proto` 文件且不需要所有文件都非常快的应用程序中最有用。
  - `LITE_RUNTIME`：协议缓冲区编译器将生成仅依赖于“lite”运行时库（`libprotobuf-lite` 而不是 `libprotobuf`）的类。
    lite 运行时比完整库小得多（大约小一个数量级），但省略了描述符和反射等某些功能。这对于在手机等受限平台上运行的应用程序特别有用。
    编译器仍将生成所有方法的快速实现，就像它在 `SPEED` 模式下一样。生成的类只会在每种语言中实现 `MessageLite` 接口，它只提供完整 `Message` 接口的方法的子集。
  
  ```protobuf
  option optimize_for = CODE_SIZE;
  ```
- `cc_enable_arenas`（文件选项）：为 C++ 生成的代码启用 [arens 分配](https://developers.google.com/protocol-buffers/docs/reference/arenas)。
- `objc_class_prefix`（文件选项）：设置 Objective-C 类前缀，该前缀添加到所有 Objective-C 生成的类和来自此 `.proto` 的枚举中。
  没有默认值。你应该按照 [Apple 的建议](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html#//apple_ref/doc/uid/TP40011210-CH10-SW4)使用介于 3-5 个大写字符之间的前缀。请注意，所有 2 个字母前缀均由 Apple 保留。
- `deprecated`（字段选项）：如果设置为 `true`，则表示该字段已弃用，不应被新代码使用。在大多数语言中，这没有实际效果。
  在 Java 中，这成为 `@Deprecated` 注释。将来，其他特定于语言的代码生成器可能会在字段的访问器上生成弃用注释，这反过来会导致在编译尝试使用该字段的代码时发出警告。
  如果该字段未被任何人使用并且你希望阻止新用户使用它，请考虑将字段声明替换为<a href="#reserved">保留</a>语句。
  
  ```protobuf
  int32 old_field = 6 [deprecated = true];
  ```

### 自定义选项 {#customoptions}

Protocol Buffers 还允许你定义和使用自己的选项。这是大多数人不需要的高级功能。
如果你确实认为需要创建自己的选项，请参阅 Proto2 语言指南了解详细信息。
请注意，创建自定义选项使用扩展，仅允许用于 proto3 中的自定义选项。

## 生成你的类 {#generating}

要生成 Java、Kotlin、Python、C++、Go、Ruby、Objective-C 或 C# 代码，
你需要使用 .proto 文件中定义的消息类型，你需要在 .proto 上运行协议缓冲区编译器协议。
如果你尚未安装编译器，请<a href="https://developers.google.com/protocol-buffers/docs/downloads">下载软件包</a>并按照 README 中的说明进行操作。
对于 Go，你还需要为编译器安装一个特殊的代码生成器插件：你可以在 GitHub 上的 [golang/protobuf](https://github.com/golang/protobuf/) 存储库中找到此插件和安装说明。

协议编译器调用如下：

```shell
protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto
```

- `IMPORT_PATH` 指定解析导入指令时要在其中查找 `.proto` 文件的目录。
  如果省略，则使用当前目录。通过多次传递 --proto_path 选项可以指定多个导入目录；他们将被按顺序搜索。
  `-I=_IMPORT_PATH_` 可以用作 `--proto_path` 的缩写形式。
- 你可以提供一个或多个输出指令：
  - `--cpp_out` 在 `DST_DIR` 中生成 C++ 代码。
    有关更多信息，请参阅 [C++ 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated)。
  - `--java_out` 在 `DST_DIR` 中生成 Java 代码。
    有关更多信息，请参阅 [Java 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/java-generated)。
  - `--kotlin_out` 在 `DST_DIR` 中生成额外的 Kotlin 代码。
    有关更多信息，请参阅 [Kotlin 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/kotlin-generated)。
  - `--python_out` 在 `DST_DIR` 中生成 Python 代码。
    有关更多信息，请参阅 [Python 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/python-generated)。
  - `--go_out` 在 `DST_DIR` 中生成 Go 代码。
    有关更多信息，请参阅 [Go 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/go-generated)。
  - `--ruby_out` 在 `DST_DIR` 中生成 Ruby 代码。
    有关更多信息，请参阅 [Ruby 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/ruby-generated)。
  - `--objc_out` 在 `DST_DIR` 中生成 Objective-C 代码。
    有关更多信息，请参阅 [Objective-C 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/objective-c-generated)。
  - `--csharp_out` 在 `DST_DIR` 中生成 C# 代码。
    有关更多信息，请参阅 [C# 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/csharp-generated)。
  - `--php_out` 在 `DST_DIR` 中生成 PHP 代码。
    有关更多信息，请参阅 [PHP 生成的代码参考](https://developers.google.com/protocol-buffers/docs/reference/php-generated)。
  作为额外的便利，如果 `DST_DIR` 以 `.zip` 或 `.jar` 结尾，编译器会将输出写入具有给定名称的单个 ZIP 格式存档文件。
  `.jar` 输出也将按照 Java JAR 规范的要求提供一个清单文件。请注意，如果输出存档已经存在，它将被覆盖；编译器不够聪明，无法将文件添加到现有存档中。
- 你必须提供一个或多个 `.proto` 文件作为输入。可以一次指定多个 `.proto` 文件。
  尽管文件是相对于当前目录命名的，但每个文件必须驻留在 `IMPORT_PATH` 之一中，以便编译器可以确定其规范名称。