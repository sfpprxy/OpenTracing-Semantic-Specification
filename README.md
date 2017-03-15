# OpenTracing语义规范(中文版)

**版本** 1.0

## 原文

[The OpenTracing Semantic Specification](https://github.com/opentracing/specification/blob/master/specification.md)

## 概述

这是一份”正式”的OpenTracing语义规范文档。由于OpenTracing是跨语言的，本文档会尽量避免提到特定语言相关的概念。也就是说，我们认为所有语言都具有类似”接口”这样的概念，并提供相关的功能。

### 版本控制策略

OpenTracing采用了 `主版本号.次版本号` 形式的版本号，但是没有 `.修订号` 。当对规范进行向后不兼容的更改时，主版本增加。次版本的增加则用于用于不间断的更改，如引入新的标准标记，日志字段或SpanContext引用类型。(你可以在这个Issue [specification#2](https://github.com/opentracing/specification/issues/2#issuecomment-261740811) 中读到更多有关此版本控制方案的信息)

## OpenTracing数据模型
OpenTracing中的**Traces**由其**Span**隐式定义。**Trace**可被认为是由一组**Span**定义的有向无环图(DAG)，在**Span**之间的被称为**References**。

以下是一个由8个**Span**构成的**Trace**的例子:

~~~
一个Trace中Span间的因果关系


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C之于Span A的关系是 `ChildOf` )
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)
~~~

有时用时间轴来显示**Traces**更容易，如下图所示:

~~~
一个Trace中Span间的时间关系


––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
~~~

每个**Span**封装了如下状态:

- 操作名称
- 开始时间戳
- 结束时间戳
- 一组零或多个键:值结构的**Span标签**(Tags)。键必须是字符串。值可以是字符串，布尔或数值类型.
- 一组零或多个**Span日志**(Logs)，其中每个都是一个键:值映射并与一个时间戳配对。键必须是字符串，值可以是任何类型。 并非所有的OpenTracing实现都必须支持每种值类型。
- 一个**SpanContext**(见下文)
- 零或多个因果相关的**Span**间的**References** (通过那些相关的**Span**的**SpanContext**)

每个SpanContext封装了如下状态:

- 任何需要跟跨进程**Span**关联的，依赖于OpenTracing实现的状态(例如Trace和Span的id)
- 键:值结构的跨进程的**Baggage Items**

### Span间的Reference

一个Span可引用零或多个因果相关的**SpanContext**。OpenTracing目前定义了两种类型的Reference: `ChildOf` 和 `FollowsFrom`。**这两种Reference类型都对父子Span间的因果关系进行了建模**。未来，OpenTracing可能会为不具因果关系的Span提供不同类型的Reference (例如批量的Span，卡在队列中的Span等)

**`ChildOf` reference**: 一个Span可以是另一个Span的子Span。在 `ChildOf` 引用中，父Span在某种程度上取决于子Span。下列情况会构成 `ChildOf` 关系:

- 在一个RPC中，代表服务端的Span可作为 `ChildOf` 代表客户端的Span
- 在一个持久化进程中，代表SQL插入的Span可作为 `ChildOf` 代表ORM save方法的Span
- 多个并发(可能是分布式)执行任务的Span可能分别各自为 `Childof` 一个合并了多个子Span结果的父Span

下列这些都是有效的具有 `ChildOf` 关系的时序图

~~~
    [-Parent Span---------]
         [-Child Span----]

    [-Parent Span--------------]
         [-Child Span A----]
          [-Child Span B----]
        [-Child Span C----]
         [-Child Span D---------------]
         [-Child Span E----]
~~~

**`FollowsFrom` reference**: 有些父Span不依赖于任何子Span的结果。这种情况下，我们仅认为子Span在因果上 `FollowsFrom` 父Span。有许多不同的 `FollowsFrom` 引用子类别，在OpenTracing的未来版本中，它们可能会被更正式地区分。

下列这些都是有效的具有 `FollowsFrom` 关系的时序图

~~~
    [-Parent Span-]  [-Child Span-]


    [-Parent Span--]
     [-Child Span-]


    [-Parent Span-]
                [-Child Span-]
~~~

## OpenTracing接口定义

OpenTracing规范中有三种相互关联的关键类型 `Tracer`，`Span` ，and `SpanContext`。接下来，我们来看一下各种类型的行为; 简单来说，每种行为分别在编程语言中对应为一个”方法”，尽管实际上可能是一组重载方法。

当我们讨论“可选”参数时，应当清楚，不同的语言有不同的方式来解释这个概念。比如说，在Go语言中我们会使用”functional Options”这个术语，而在Java中我们会使用builder模式。

### `Tracer`

`Tracer` 接口创造 `Span` 并且能够跨进程地 `Inject` (序列化)和 `Extract` (反序列化)。严格来说，它应具有以下能力:

#### 启动一个新的 `Span`

必须参数

- **操作名称**，一个人工可读的字符串，它简洁地表示由Span完成的工作 (例如，RPC方法名称，函数名称或一个较大的计算任务中的阶段的名称)。**操作名称应该用泛化的字符串形式标识出一个Span实例.** 也就是说，`get_user` 比 `get_user/314159` 好.

比如说这里有几个**操作名称**，用于得到一个虚构的账户信息:

| 操作名称              | 建议                                       |
| :---------------- | :--------------------------------------- |
| `get`             | 太宽泛                                      |
| `get_account/792` | 太具体                                      |
| `get_account`     | 刚刚好，`account_id=792` 可作为一个不错的 **`Span` 标签** |

可选参数

- 零或多个与 `SpanContext` 相关的**references**，尽可能包括 `ChildOf` 和 `FollowFrom` 引用类型的信息
- **开始时间戳**，如果没有的话，默认使用现实时间(walltime)
- 零或多个**标签**

**返回值**是一个已经启动还未结束的Span实例

#### 注入(Inject) `SpanContext`到载体中

必须参数

- `SpanContext`实例
- **格式描述符(format descriptor)**(通常但不一定是字符串常量)，告诉Tracer的实现如何在载体对象中对SpanContext进行编码
- **载体(carrier)**，其类型由格式描述符指定。Tracer的实现将根据**格式描述**对此载体对象中的SpanContext进行编码

#### 从载体中抽取(Extract) `SpanContext`

必须参数

- **格式描述符(format descriptor)**，告诉Tracer的实现如何在载体对象中对SpanContext进行解码
- **载体**，其类型由**格式描述符**指定。 Tracer的实现将根据**格式描述**对此载体对象中的SpanContext进行解码

**返回值**是一个SpanContext实例，适合作为通过Tracer启动Span时的reference

#### 注意: Injection和Extraction所需的格式

Injection和Extraction都依赖于一个可扩展的格式描述符参数，其指定了相关联的”载体”类型以及 `SpanContext` 是如何被编码的。Tracer实现必须支持下列所有格式。

- **文本映射(Text Map):** 任意的无字符编码限制的string-string型键值结构
- **HTTP Headers:** string-string型的键值结构在HTTP headers( [RFC 7230](https://tools.ietf.org/html/rfc7230#section-3.2.4) )中也适用。在实操中，由于HTTP header很容易被各种不同的方式处理， 强烈建议在Tracer的实现中使用有限的键空间和保守的转意值
- **二进制:** 代表了 `SpanContext` 的任意二进制数据

### `Span`

除了获取 `Span` 的 `SpanContext` 的方法之外，下面任何方法都不能在 `Span` 完成后被调用。

#### 获取 `Span` 的 `SpanContext` 

无需参数.

返回值是 `Span` 的 `SpanContext` 。返回值甚至可能在 `Span` 结束后被使用。

#### 重写操作名

必须参数

- 新的**操作名**，当 `Span` 启动时取代旧内容

#### 结束 `Span`

可选参数

- 显式地添加Span的结束时间戳，如果没有这个参数则会隐式的添加现实时间(walltime)
  除了获取Span的SpanContext的方法之外，任何Span对象的方法都不能在其完成后被调用。

#### 设置 `Span` 标签

必须参数

- 标签的键，必须是字符串
- 标签的值，必须是字符串，布尔值，数值类型其中之一

  注意OpenTracing项目文档中已经为一些”[标准标签](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)”规定了语义。

#### `记录(Log)` 结构化的数据

必须参数

- 一到多个键值对，键必须是字符串，值可以是任意类型。有些OpenTracing的实现可以处理更多的日志的值。

  可选参数

- 显式时间戳。该参数必须落在当前Span的开始与结束时间戳之间。

注意OpenTracing项目文档中已经为一些“[日志的标准键](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)“ 规定了语义。

#### 设置(set) baggage item

Baggage item是对应于某个 `Span` 及其 `SpanContext` ，以及**所有直接或间接引用自本地(local) `Span` 的 `Span` 的键值对**。也就是说，baggage items是与其trace一起传播的。

Baggage item为全栈式的OpenTracing集成提供了强大的功能(比如在移动App上使用时，它可以一路追踪数据直至储存系统的深度)，不过使用这些功能时要当心。

每个键值都会被拷贝到每一个本地(local)及远程的子Span，这可能导致巨大的网络和CPU开销。

必须参数

- **baggage键**，字符串
- **baggage值**，字符串

#### 获取(get) baggage item

必须参数

- **baggage键**，字符串

**返回值**是对应的**baggage值**，或是一个表示没有baggage值的一个量

### `SpanContext`

在一般所指的OpenTracing层面上，`SpanContext` 更像是一个概念而不是一种实用功能。也就是说，这对OpenTracing的实现来说提供一层薄的接口是至关重要的。当启动一个 `Span` 或者注入/提取(injecting/extracting) trace时，多数OpenTracing用户仅通过**reference**与 `SpanContext` 进行交互。

在OpenTracing中，`SpanContext` 被强制设定为**不可变的**，以应对在 `Span` 结束时和引用(reference)时产生的复杂的生命周期问题。

#### 遍历所有的baggage item

不同语言对此有不同的建模方式，不过给出一个 `SpanContext` 实例，语义上还是应该能让调用者在短时间内遍历baggage items。

### `NoopTracer`

所有语言的OpenTracing API必须提供某种 `NoopTracer` 实现，可用作标记控制(flag-control)或者进行一些用于测试的无害注入等。某些情况下(比如Java)，`NoopTracer` 可能在它自己的制品包(packaging artifact)中。

### 可选的API元素

有些语言提供了在单进程中用来传递活动的 `Span` 和 `SpanContext` 的工具。比如，`opentracing-go` 提供了在Go的 `context.Context` 机制中，用来set和get活动 `Span` 的帮助函数(helpers)。
