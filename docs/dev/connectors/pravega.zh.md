---
title: "Pravega Connector"
nav-title: Pravega
nav-parent_id: connectors
nav-pos: 6
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* This will be replaced by the TOC
{:toc}

## Pravega 介绍
[Pravega](http://pravega.io/) 是一款开源、高性能、持久化、支持弹性伸缩、强一致性的针对持续无界数据的流存储产品。

首先介绍一些 Pravega 的基础概念和术语。

Pravega 构建在 *Stream* 的基础之上。 一个 stream 由一个或多个字节序列组成。 构成 stream 的单元被称为 *segment*。
一个 *segment* 是一个无界的字节序列。 一个 stream 在任何时间可以包含一个或多个 segment ，segment 可以动态地进行自动伸缩。
为了避免 stream 重名，保证不同应用的隔离性，每一个 Stream 会包含在一个 *scope* 的命名空间下。

*Event* 是 Pravega 提供的数据抽象。 Event 写入时会被序列化成为字节流，并添加到 segment 之中。 Pravega 利用用户可指定的 *routing key*
指定 event 和 segment 之间的对应关系，使得 events 能够在 open segment （由 Pravega 控制面管理）中分布。 

包括 Flink 作业在内的应用程序使用 reader group 读取 Pravega stream 中的数据. 在同一个 reader group 的 reader 能够并行地
读取 segment 中的数据。 Reader 对于 segment 的负载分配由内部实现的状态同步机制 *state synchronizer* 进行协调。这样的协调机制
对应用透明，因此应用无需对于 segment 集合的动态变化进行处理。 同时，*state synchronizer* 也衍生出了 *Checkpoint* 的概念，
记录了 reader group 的读取进度，并且通过与 Flink checkpoint 机制的挂钩成为实现与Flink端到端恰好一次语义的基础。 

Checkpoint 通过 *stream cut* —— 不同的 segment 上的偏移量构成。 除了 Checkpoint 机制之外，stream cuts 还可以用以指定读取数据的
起始和终止位置。为了能够简化事件流上的计算的恢复和限定，Pravega API 允许 reader 直接使用 *stream cut*。

## Pravega 连接器
Pravega 连接器包括了 source 和 sink 的实现，能够让 Flink 任务使用 Pravega 作为数据的输入和输出。 以下章节的例子将会阐述
Flink 应用如何使用 Pravega source 和 sink。

[Pravega Samples](https://github.com/pravega/pravega-samples) 中也有更多的应用实例。

### 功能与亮点
- 支持端到端的恰好一次语义的处理流水线
- 与 Flink checkpoint 和 savepoint 的无缝集成
- 支持高吞吐、低延时的并行读写
- 对批、流处理两种使用场景提供了不同的 Table API

### 用法
使用前，需要选择适当的依赖，并且在工程里添加，具体如下：

<div class="codetabs" markdown="1">
<div data-lang="Maven" markdown="1">
{% highlight xml %}
<!-- Before Pravega 0.6 -->
<dependency>
  <groupId>io.pravega</groupId>
  <artifactId>pravega-connectors-flink_2.12</artifactId>
  <version>0.5.1</version>
</dependency>

<!-- Pravega 0.6 and After -->
<dependency>
  <groupId>io.pravega</groupId>
  <artifactId>pravega-connectors-flink-1.9_2.12</artifactId>
  <version>0.7.0</version>
</dependency>
{% endhighlight %}
</div>
<div data-lang="Gradle" markdown="1">
{% highlight gradle %}
// Before Pravega 0.6
dependencies {
  compile "io.pravega:pravega-connectors-flink_2.12:0.5.1"
}

// Pravega 0.6 and After
dependencies {
  compile "io.pravega:pravega-connectors-flink-1.9_2.12:0.7.0"
}
{% endhighlight %}
</div>
</div>

{% info %} 在上例中,

`1.9` 是 Flink 的主次版本号。

`2.12` 为 Scala 版本, 目前仅支持 Scala 2.12。

`0.7.0` 是 Pravega 版本号，用户可以在 [GitHub Releases page](https://github.com/pravega/flink-connectors/releases) 中查询
最新的发布版本信息。

Pravega 从 0.6.0 版本起， Pravega 连接器开始支持多个 Flink 版本。
Pravega 连接器的发布版本支持最近*三*个 Flink 主版本，具体支持的 Flink 版本信息可参考 Pravega 连接器的发布公告.

## 部署 Pravega
Pravega 可以在不同的环境中以多种方式进行部署。具体步骤可参考
[Pravega deployment](http://pravega.io/docs/latest/deployment/deployment/)。


## 准备工作

### 配置信息
`PravegaConfig` 类可以为连接器建立 Pravega 的上下文。它可以自动从 _环境变量_、 _系统配置_ 以及 _程序参数_ 中获取信息进行自动配置。

`PravegaConfig` 包含如下配置:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">配置项</th>
      <th class="text-left">环境变量名</th>
      <th class="text-left">系统配置名</th>
      <th class="text-left">程序参数名</th>
      <th class="text-left">默认值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Controller URI</td>
      <td>PRAVEGA_CONTROLLER_URI</td>
      <td>pravega.controller.uri</td>
      <td>--controller</td>
      <td>tcp://localhost:9090</td>
    </tr>
    <tr>
      <td>默认 Scope</td>
      <td>PRAVEGA_SCOPE</td>
      <td>pravega.scope</td>
      <td>--scope</td>
      <td>-</td>
    </tr>
    <tr>
      <td>认证信息</td>
      <td>-</td>
      <td>-</td>
      <td>-</td>
      <td>-</td>
    </tr>
    <tr>
      <td>主机名校验</td>
      <td>-</td>
      <td>-</td>
      <td>-</td>
      <td>true</td>
    </tr>
  </tbody>
</table>

推荐可用 `ParameterTool` 类从程序参数中构造 `PravegaConfig` 实例:
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ParameterTool params = ParameterTool.fromArgs(args);
PravegaConfig config = PravegaConfig.fromParams(params);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val params = ParameterTool.fromArgs(args)
val config = PravegaConfig.fromParams(params)
{% endhighlight %}
</div>
</div>

### 序列化
用户可以直接使用Flink 的序列化/反序列化接口（例如， `SimpleStringSchema`）。

另一种常见的使用场景为，数据由上游非 Flink 应用，而是由 Pravega 客户端产生，用户使用 Flink 对 Pravega stream 中的数据进行处理。
Pravega 客户端会自定义实现序列化接口
[`io.pravega.client.stream.Serializer`](http://pravega.io/docs/latest/javadoc/clients/io/pravega/client/stream/Serializer.html)
以进行不同格式数据的序列化。
 `Serializer` 的不同实现可以由连接器提供的适配器进行转化，从而直接在 Flink 应用中使用。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import io.pravega.client.stream.impl.JavaSerializer;
...
DeserializationSchema<MyEvent> adapter = new PravegaDeserializationSchema<>(
    MyEvent.class, new JavaSerializer<MyEvent>());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
import io.pravega.client.stream.impl.JavaSerializer
...
val adapter = new PravegaDeserializationSchema[MyEvent](
    classOf[MyEvent], new JavaSerializer[MyEvent]())
{% endhighlight %}
</div>
</div>

## Pravega Reader

### 流处理 API
连接器提供 `FlinkPravegaReader` 类用以实时消费一个或多个 Pravega streams 中的数据，示例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Enable checkpoint to make state fault tolerant
env.enableCheckpointing(...);

// Define the Pravega configuration
PravegaConfig config = PravegaConfig.fromParams(params);

// Define the event deserializer
DeserializationSchema<MyClass> deserializer = ...

// Define the data stream
FlinkPravegaReader<MyClass> pravegaSource = FlinkPravegaReader.<MyClass>builder()
    .forStream(...)
    .withPravegaConfig(config)
    .withDeserializationSchema(deserializer)
    .build();
DataStream<MyClass> stream = env.addSource(pravegaSource);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

// Enable checkpoint to make state fault tolerant
env.enableCheckpointing(...)

// Define the Pravega configuration
val config = PravegaConfig.fromParams(params)

// Define the event deserializer
val deserializer = ...

// Define the data stream
val pravegaSource = FlinkPravegaReader
    .builder[MyClass]()
    .forStream(...)
    .withPravegaConfig(config)
    .withDeserializationSchema(deserializer)
    .build()
    
val stream = env.addSource(pravegaSource)
{% endhighlight %}
</div>
</div>

### 批处理 API
类似地, 连接器提供 `FlinkPravegaInputFormat` 类以批处理高吞吐方式读取 stream 中的数据，并返回 `DataSet`。示例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Define the input format based on a Pravega stream
FlinkPravegaInputFormat<EventType> inputFormat = FlinkPravegaInputFormat.<EventType>builder()
    .forStream(...)
    .withPravegaConfig(config)
    .withDeserializationSchema(deserializer)
    .build();

DataSource<EventType> dataSet = env.createInput(inputFormat, TypeInformation.of(EventType.class))
                                   .setParallelism(2);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// Define the input format based on a Pravega stream
val inputFormat = FlinkPravegaInputFormat.builder[EventType]()
    .forStream(...)
    .withPravegaConfig(config)
    .withDeserializationSchema(deserializer)
    .build()
val dataSet = env.createInput(inputFormat, TypeInformation.of(classOf[EventType])).setParallelism(2)
{% endhighlight %}
</div>
</div>

## Pravega Writer

### 流处理 API
连接器提供 `FlinkPravegaWriter` 类用以实时将事件流写入 Pravega stream。示例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Define the Pravega configuration
PravegaConfig config = PravegaConfig.fromParams(params);

// Define the event serializer
SerializationSchema<MyClass> serializer = ...

// Define the event router for selecting the Routing Key (lambda expression can be used for simplification)
PravegaEventRouter<MyClass> router = ...

// Define the sink function
FlinkPravegaWriter<MyClass> pravegaSink = FlinkPravegaWriter.<MyClass>builder()
   .forStream(...)
   .withPravegaConfig(config)
   .withSerializationSchema(serializer)
   .withEventRouter(router)
   .withWriterMode(PravegaWriterMode.EXACTLY_ONCE)
   .build();

DataStream<MyClass> stream = ...
stream.addSink(pravegaSink);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

// Define the Pravega configuration
val config = PravegaConfig.fromParams(params)

// Define the event serializer
val serializer = ...

// Define the event router for selecting the Routing Key (lambda expression can be used for simplification)
val router = ...

// Define the sink function
val pravegaSink = FlinkPravegaWriter.builder[MyClass]()
    .forStream(...)
    .withPravegaConfig(config)
    .withSerializationSchema(serializer)
    .withEventRouter(router)
    .withWriterMode(PravegaWriterMode.EXACTLY_ONCE)
    .build()

val stream: DataStream[MyClass] = ...
stream.addSink(pravegaSink)
{% endhighlight %}
</div>
</div>

#### 写模式
写模式与写入 Pravega Stream 的语义保证相关，包括以下三种：

1. **Best-effort** - 所有的写失败会被忽略，允许数据丢失。
2. **At-least-once** - 所有的 event 都会在 Pravega 中保存，会因为失败重试产生重复数据。
3. **Exactly-once** - 所有的 event 都会结合 Flink checkpoint 机制以事务性的写入方式在 Pravega 中保存。
更多细节可以进一步阅读 Flink 端到端恰好一次的介绍
[博客](https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html).

默认开启 _At-least-once_ 选项，可以使用 `.withWriterMode(...)` 方法更改写模式。

### 批处理 API
类似地, 连接器提供 `FlinkPravegaOutputFormat` 类支持批处理的写入操作。可以将 Pravega Stream 作为 `DataSet` 的 sink。 
示例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// Define the output format based on a Pravega stream
FlinkPravegaOutputFormat<EventType> outputFormat = FlinkPravegaOutputFormat.<EventType>builder()
    .forStream(...)
    .withPravegaConfig(config)
    .withSerializationSchema(serializer)
    .withEventRouter(router)
    .build();

DataSet<EventType> dataSet = ...
dataSet.output(outputFormat);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// Define the output format based on a Pravega stream
val outputFormat = FlinkPravegaOutputFormat.builder[EventType]()
    .forStream(...)
    .withPravegaConfig(config)
    .withSerializationSchema(serializer)
    .withEventRouter(router)
    .build()

val dataSet: DataSet[EventType] = ...
dataSet.output(outputFormat)
{% endhighlight %}
</div>
</div>

更多例如 Table API、事件时间排序、watermark 支持、指标收集等详细功能敬请阅读完整文档
[Pravega Connector Document](http://pravega.io/docs/latest/connectors/flink-connector/)
