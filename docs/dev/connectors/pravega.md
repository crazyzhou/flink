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

## Pravega Introduction
[Pravega](http://pravega.io/) is an open-source, high-performance, durable, elastic and strongly-consistent
stream storage for continuous and unbounded data.

Here are some need-to-know Pravega terminologies.

Pravega builds on the concept of *Stream*. A stream comprises one or more sequences of bytes.
The unit of a stream is the *segment*. A *segment* is an unbounded sequence of bytes. A stream
can have one or more segments at any time, and the set of segments is allowed to change
dynamically with auto-scaling. Streams are grouped into *scopes* to avoid name collisions across
applications.

One of the main APIs that Pravega provides is the *event*. Events are serialized into bytes and
appended to segments. Pravega uses *routing keys* to map events to segments, they are
essentially sharded across open segments, and Pravega clients learn about open segments from
the control plane of Pravega.

Applications, including Flink jobs, read from a Pravega stream using *reader groups*. The readers
in a group share the workload of segments and coordinate the assignment of segments using a
Pravega feature that enables the coordination of state called *state synchronizer*. That coordination
is transparent to the application so that application does not have to handle dynamic changes to
the set of segments. This coordination also enables the generation of *checkpoints*, which are
essential to enable exactly-once semantics end-to-end with Flink.

The foundation of checkpoints is an abstraction called *stream cut*: a collection of offsets
spanning one or more segments. Checkpointing constitutes one of the main use cases of stream cuts,
but stream cuts can additionally be used to set a position to begin reading or a position to end
reading in a stream. To be able to resume or bound the computation over a stream of events, the
Pravega API enables readers to obtain *stream cut* objects independent of checkpoints.

## Pravega connector
The Pravega connector comprises source and sink implementations that enable jobs to use Pravega as an
input of stream data and to store the output of jobs, respectively. The examples we show below in this
document illustrate how an application sets it up to use the Pravega source and sink.

See also the [Pravega Samples](https://github.com/pravega/pravega-samples) repository for more examples.

### Features & Highlights
- Supporting end-to-end exactly-once processing pipelines
- Seamless integration with Flink's checkpoints and savepoints.
- Parallel Readers and Writers supporting high throughput and low latency processing.
- Table API support to access Pravega Streams for both Batch and Streaming use case.

### Dependency
To use this connector, please pick the proper dependency and add it to your project:

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

{% info %} As in the above example,

`1.9` is the Flink Major-Minor version which should be provided to find the right artifact.

`2.12` is the Scala version, now only 2.12 is supported.

`0.7.0` is the Pravega version, you can find the latest release on
the [GitHub Releases page](https://github.com/pravega/flink-connectors/releases).

Since Pravega 0.6.0, the connector supports multiple Flink versions.
Pravega connector maintains compatibility for the *three* most recent major versions of Flink.
See the Pravega connector release notes for supported versions.

## Deploying Pravega
There are multiple options provided for running Pravega in different environments.
Follow the instructions from [Pravega deployment](http://pravega.io/docs/latest/deployment/deployment/)
to deploy a Pravega cluster.

## Preparation

### Configurations
A top-level config object, `PravegaConfig`, is provided to establish a Pravega context for the connector.
The config object can automatically configure itself from _environment variables_, _system properties_ and _program arguments_.

`PravegaConfig` information is given below:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Setting</th>
      <th class="text-left">Environment Variable</th>
      <th class="text-left">System Property</th>
      <th class="text-left">Program Argument</th>
      <th class="text-left">Default Value</th>
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
      <td>Default Scope</td>
      <td>PRAVEGA_SCOPE</td>
      <td>pravega.scope</td>
      <td>--scope</td>
      <td>-</td>
    </tr>
    <tr>
      <td>Credentials</td>
      <td>-</td>
      <td>-</td>
      <td>-</td>
      <td>-</td>
    </tr>
    <tr>
      <td>Hostname Validation</td>
      <td>-</td>
      <td>-</td>
      <td>-</td>
      <td>true</td>
    </tr>
  </tbody>
</table>

The recommended way to create an instance of `PravegaConfig` is to pass an instance of `ParameterTool` to `fromParams`:
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

### Serialization
Flink's serialization/deserialization interfaces(eg. `SimpleStringSchema`) can be used directly.

Another common scenario is to process Pravega stream data produced by a non-Flink application.
The Pravega client library used by such applications defines the
[`io.pravega.client.stream.Serializer`](http://pravega.io/docs/latest/javadoc/clients/io/pravega/client/stream/Serializer.html)
interface for working with event data.
The implementations of `Serializer` directly in a Flink program via built-in adapters can be used:
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

### Streaming API
This connector provides a `FlinkPravegaReader` class to consume events from one or multiple Pravega streams in real-time.
See below for an example:
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

### Batch API
Similarly, a `FlinkPravegaInputFormat` class is offered for batch processing. It reads events of a stream as a `DataSet`.
See below for an example:
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

### Streaming API
This connector provides a `FlinkPravegaWriter` class to write events into a Pravega stream in real-time.
See below for an example:
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

#### Writer Modes
Writer modes relate to guarantees about the persistence of events emitted by the sink to a Pravega Stream.
The writer supports three writer modes:

1. **Best-effort** - Any write failures will be ignored hence there could be data loss.
2. **At-least-once** - All events are persisted in Pravega. Duplicate events
are possible, due to retries or in case of failure and subsequent recovery.
3. **Exactly-once** - All events are persisted in Pravega using a transactional approach integrated with the Flink checkpointing feature.
For more details, please refer to this 
[blog](https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html) introducing end-to-end exactly once.

By default, the _At-least-once_ option is enabled and `.withWriterMode(...)` option can be used to override the value.

### Batch API
Similarly, a `FlinkPravegaOutputFormat` class is offered for batch processing.
It can be supplied to make a Pravega Stream as a sink of a `DataSet`.
See below for an example:
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

Details on more functionality such as Table API, event time ordering, watermark support and metrics can be found in the
[Pravega Connector Document](http://pravega.io/docs/latest/connectors/flink-connector/)
