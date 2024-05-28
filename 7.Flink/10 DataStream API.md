#### DataStream API

##### 1. 执行环境（Execution Environment）

```
Flink 程序可以在各种上下文环境中运行：可以在本地 JVM 中执行程序，也可以提交到远程集群上运行。
不同的环境，代码的提交运行的过程会有所不同。这就要求在提交作业执行计算时， 首先必须获取当前 Flink 的运行环境，从而建立起与 Flink 框架之间的联系。只有获取了环境上下文信息，才能将具体的任务调度到不同的TaskManager 执行
```

###### 1.1 创建执行环境

**getExecutionEnvironment**

* 如果程序是独立运行的，就返回一个本地执行环境
* 如果是创建了jar包，然后从命令行调用它并提交到集群执行，那么就返回集群的执行环境

**createLocalEnvironment**

```
StreamExecutionEnvironment localEnv = StreamExecutionEnvironment.createLocalEnvironment();

这个方法返回一个本地执行环境。可以在调用时传入一个参数，指定默认的并行度；如果不传入，则默认并行度就是本地的CPU核心数
```

**createRemoteEnvironment**

```
StreamExecutionEnvironment remoteEnv = StreamExecutionEnvironment
 .createRemoteEnvironment(
 "host", // JobManager 主机名
 1234, // JobManager 进程端口号
 "path/to/jarFile.jar" // 提交给 JobManager 的 JAR 包
);
```

###### 1.2 执行模式

```
// 批处理环境
ExecutionEnvironment batchEnv = ExecutionEnvironment.getExecutionEnvironment();
// 流处理环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

```

从1.12.0版本起，Flink实现了API上的流批统一。DataStreamAPI新增了一个重要特性：**可以支持不同的“执行模式”（executionmode），通过简单的设置就可以让一段Flink程序在流处理和批处理之间切换。**这样一来，DataSetAPI也就没有存在的必要

* **流执行模式（STREAMING）**

这是DataStreamAPI最经典的模式，一般用于需要持续实时处理的无界数据流。默认情况下，程序使用的就是STREAMING执行模式

* **批执行模式（BATCH）**

专门用于批处理的执行模式, 这种模式下，Flink 处理作业的方式类似于 MapReduce 框架。 对于不会持续计算的有界数据，用这种模式处理会更方便

* **自动模式（AUTOMATIC）**

在这种模式下，将由程序根据输入数据源是否有界，来自动选择执行模式

```
BATCH模式的配置方法；
由于 Flink 程序默认是 STREAMING 模式，这里重点介绍一下 BATCH 模式的配置
1. 通过命令行配置
bin/flink run -Dexecution.runtime-mode=BATCH ...
2. 通过代码配置(不推荐-灵活性差)
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setRuntimeMode(RuntimeExecutionMode.BATCH);

```

###### 1.3 触发程序执行

有了执行环境，就可以构建程序的处理流程了：基于环境读取数据源，进而进行各种转换操作，最后输出结果到外部系统。
需要注意的是，写完输出（sink）操作并不代表程序已经结束。因为当 main()方法被调用时，其实只是定义了作业的每个执行操作，然后添加到数据流图中；这时并没有真正处理数据,因为数据可能还没来。Flink 是由事件驱动的，只有等到数据到来，才会触发真正的计算，这也被称为“延迟执行”或“懒执行”（lazy execution）。
所以需要显式地调用执行环境的 execute()方法，来触发程序执行。execute()方法将一直等待作业完成，然后返回一个执行结果（JobExecutionResult）

```java
env.execute();
```

##### 2 Source

```java
DataStream<String> stream = env.addSource(...);
```

###### 2.1 从集合中读取

```java
 env.fromCollection()
```

###### 2.2 从文本读取数据

```java
DataStreamSource<String> Stream = env.readTextFile("input/clicks.txt");
```

说明:

- 参数可以是目录，也可以是文件；
- 路径可以是相对路径，也可以是绝对路径；
- 相对路径是从系统属性user.dir获取路径: idea 下是project的根目录, standalone模式下是集群节点根目录；
- 也可以从hdfs目录下读取, 使用路径 hdfs://..., 由于Flink没有提供hadoop相关依赖, 需要pom中添加相关依赖:

```java
<dependency>
 <groupId>org.apache.hadoop</groupId>
 <artifactId>hadoop-client</artifactId>
 <version>2.7.5</version>
 <scope>provided</scope>
</dependency>
```

###### 2.3 从Socket读取数据

```java
DataStreamSource<String> Stream4 = env.socketTextStream("hadoop102", 7777);
```

###### 2.4 从kafka读取数据

那对于真正的流数据，实际项目应该怎样读取呢？ Kafka 作为分布式消息传输队列，是一个高吞吐、易于扩展的消息系统。而消息队列的传输方式，恰恰和流处理是完全一致的。所以可以说Kafka和Flink天生一对，是当前处理流式数据的双子星。在如今的实时流处理应用中，由 Kafka 进行数据的收集和传输，Flink 进行分析计算，这样的架构已经成为众多企业的首选;
略微遗憾的是，Kafka的连接比较复杂，Flink内部并没有提供预实现的方法。所以只能采用通用的addSource方式、实现一个SourceFunction 。好在Kafka与Flink确实是非常契合，所以Flink官方提供了连接工具flink-connector-kafka，直接实现了一个消费者FlinkKafkaConsumer，它就是用来读取Kafka 数据的SourceFunction。 所以想要以Kafka作为数据源获取数据，只需要引入Kafka连接器的依赖。Flink官方提供的是一个通用的Kafka连接器，它会自动跟踪最新版本的Kafka客户端。目前最新版本只支持0.10.0版本以上的Kafka，使用时可以根据自己安装的Kafka版本选定连接器的依赖版本。这里需要导入的依赖如下。

```java
<dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
</dependency>
```

```java
package com.kunan.StreamAPI.Source;
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import java.util.ArrayList;
import java.util.Properties;
public class SourceTest {
    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行度1
        env.setParallelism(1);
		//从Kafka中读取数据
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "hadoop102:9092");
        properties.setProperty("group.id", "test-consumer-group");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");
        DataStreamSource<String> kafkaStream = env.addSource(new FlinkKafkaConsumer<String>("clicks", new SimpleStringSchema(), properties));
        kafkaStream.print("Kafka");
        env.execute();
    }
}
```

注意：

- 确保集群已安装Kafka；
- Kafka是依赖Zookeeper的，所以两者都得安装才可测试

创建FlinkKafkaConsumer时需要传入三个参数：

- 第一个参数 topic，定义了从哪些主题中读取数据。可以是一个topic，也可以是topic列表，还可以是匹配所有想要读取的topic的正则表达式。当从多个topic中读取数据时，Kafka连接器将会处理所有topic的分区，将这些分区的数据放到一条流中去。
- 第二个参数是一个DeserializationSchema或者KeyedDeserializationSchema。Kafka 消息被存储为原始的字节数据，所以需要反序列化成Java或者 Scala对象。上面代码中使用的SimpleStringSchema，是一个内置的DeserializationSchema，它只是将字节数组简单地反序列化成字符串。DeserializationSchema和KeyedDeserializationSchema是公共接口，所以也可以自定义反序列化逻辑。
- 第三个参数是一个Properties对象，设置了Kafka客户端的一些属性。

###### 2.5 自定义source

大多数情况下，前面的数据源已经能够满足需要。但是凡事总有例外，如果遇到特殊情况， 想要读取的数据源来自某个外部系统，而flink既没有预实现的方法、也没有提供连接器，该怎么办呢？那就只好自定义实现SourceFunction了。 接下来创建一个自定义的数据源，实现SourceFunction接口。
主要重写两个关键方法： **run()和 cancel()**。

- run()方法：使用运行时上下文对象（SourceContext）向下游发送数据；
- cancel()方法：通过标识位控制退出循环，来达到中断数据源的效果。
- 代码如下：

```java
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import java.util.Calendar;
import java.util.Random;
public class ClickSource implements SourceFunction<Event> {
    //声明一个标志位
    private Boolean running = true;
    @Override
    p所以如果想要自定义并行的数据源的话，需要使用ParallelSourceFunction，示例程序如下    while (running){
            String user = users[random.nextInt(users.length)];
            String url = urls[random.nextInt(urls.length)];
            Long timestamp = Calendar.getInstance().getTimeInMillis();
            sourceContext.collect(new Event(user,url,timestamp));
            Thread.sleep(1000);
        }
    }
    @Override
    public void cancel() {
            running = false;
    }
}
```

```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import java.util.Random;
public class SourceCustomTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行度为1
        env.setParallelism(1);
        env.setParallelism(4);
        DataStreamSource<Event> CustomSource = env.addSource(new ClickSource());
        CustomSource.print();
        env.execute();
    }
}
```

所以如果想要自定义并行的数据源的话，需要使用ParallelSourceFunction，示例程序如下

```java
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import java.util.Random;
public class SourceCustomTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);
        DataStreamSource<Integer> CustomSource = env.addSource(new ParallelCustomSource()).setParallelism(2);
        CustomSource.print();
        env.execute();
    }
    public static class ParallelCustomSource implements ParallelSourceFunction<Integer>{
        private Boolean running  = true;
        private Random random = new Random();
        @Override
        public void run(SourceContext<Integer> ctx) throws Exception {
            while (running){
                ctx.collect(random.nextInt());
            }
        }
        @Override
        public void cancel() {
            running = false;
        }
    }
}
```

##### 3 Flink 支持的数据类型

```
为什么会出现“不支持”的数据类型呢？因为Flink作为一个分布式处理框架，处理的是以数据对象作为元素的流。如果用水流来类比，那么要处理的数据元素就是随着水流漂动的物体。在这条流动的河里，可能漂浮着小木块，也可能行驶着内部错综复杂的大船。要分布式地处理这些数据，就不可避免地要面对数据的网络传输、状态的落盘和故障恢复等问题，这 就需要对数据进行序列化和反序列化。小木块是容易序列化的；而大船想要序列化之后传输，就需要将它拆解、清晰地知道其中每一个零件的类型。 为了方便地处理数据，Flink有自己一整套类型系统。Flink使用“类型信息” （TypeInformation）来统一表示数据类型。TypeInformation类是Flink中所有类型描述符的基类。 它涵盖了类型的一些基本属性，并为每个数据类型生成特定的序列化器、反序列化器和比较器
```

###### 3.1 **Flink支持的数据类型**

对于常见的Java和Scala数据类型，Flink都是支持的。Flink在内部，Flink对支持不同的类型进行了划分，这些类型可以在Types工具类中找

1. 基本类型:所有Java 基本类型及其包装类，再加上Void、String、Date、BigDecimal和BigInteger
2. 数组类型:包括基本类型数组（PRIMITIVE_ARRAY）和对象数组(OBJECT_ARRAY)
3. 复合数据类型：

- Java元组类型（TUPLE）：这是Flink内置的元组类型，是Java API的一部分。最多25个字段，也就是从Tuple0~Tuple25，不支持空字段;
- Scala样例类及Scala元组：不支持空字段;
- 行类型（ROW）：可以认为是具有任意个字段的元组,并支持空字段;
- POJO：Flink自定义的类似于Java bean模式的类;

4. 辅助类型：Option、Either、List、Map 等
5. 泛型类型：

```
Flink支持所有的Java类和Scala类。不过如果没有按照上面POJO类型的要求来定义，就会被Flink当作泛型类来处理。Flink会把泛型类型当作黑盒，无法获取它们内部的属性；它们也不是由Flink本身序列化的，而是由Kryo序列化的。 在这些类型中，元组类型和 POJO 类型最为灵活，因为它们支持创建复杂类型。而相比之下，POJO 还支持在键（key）的定义中直接使用字段名，这会让代码可读性大大增加。所以，在项目实践中，往往会将流处理程序中的元素类型定为Flink的POJO类型。
```

**Flink对POJO类型的要求**

- 类是公共的（public）和独立的（standalone，也就是说没有非静态的内部类）；
- 类有一个公共的无参构造方法；
- 类中的所有字段是public且非final的；或者有一个公共的getter和setter方法，这些方法需要符合Java bean的命名规范。之前的自定义source，就是创建的符合 Flink POJO 定义的数据类型

###### 3.2 类型提示

Flink还具有一个类型提取系统，可以分析函数的输入和返回类型，自动获取类型信息， 从而获得对应的序列化器和反序列化器。但是，由于Java中泛型擦除的存在，在某些特殊情况下（比如Lambda表达式中），自动提取的信息是不够精细的——只告诉Flink当前的元素由 “船头、船身、船尾”构成，根本无法重建出“大船”的模样；这时就需要显式地提供类型信 息，才能使应用程序正常工作或提高其性能。
为了解决这类问题，Java API提供了专门的“类型提示”（type hints）。 之前的word count流处理程序，在将String类型的每个词转换成（word， count）二元组后，就明确地用returns指定了返回的类型。因为对于 map 里传入的Lambda表达式，系统只能推断出返回的是Tuple2类型，而无法得到 Tuple2<String，Long>。只有显式地告诉系统当前的返回类型，才能正确地解析出完整数据。

```java
.map(word -> Tuple2.of(word, 1L))
.returns(Types.TUPLE(Types.STRING, Types.LONG))
```

这是一种比较简单的场景，二元组的两个元素都是基本数据类型。那如果元组中的一个元素又有泛型，如何处理?
Flink专门提供了TypeHint类，它可以捕获泛型的类型信息，并且一直记录下来，为运行时提供足够的信息。同样可以通过.returns()方法，明确地指定转换之后的DataStream里元素的类型

```java
returns(new TypeHint<Tuple2<Integer,SomeType>>(){})
```

##### 4 转换

###### 4.1 基本转换算子

###### 4.1.1 映射（map）

map是非常熟悉的大数据操作算子，主要用于将数据流中的数据进行转换，形成新的数据流。简单来说，就是一个“一一映射”，消费一个元素就产出一个元素

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
public class TransformMapTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L)
        );
        //进行转计算,提取usr字段
        //1.使用自定义类实现MapFunction接口
        SingleOutputStreamOperator<String> result1 = Stream.map(new MyMapper());
        //2.使用匿名类实现MapFunction接口
        SingleOutputStreamOperator<String> result2 = Stream.map(new MapFunction<Event, String>() {
            @Override
            public String map(Event event) throws Exception {
                return event.user;
            }
        });
        //3.传入Lambda表达式  对于类里面只有一个方法的接口 可以使用Lambda表达式
        SingleOutputStreamOperator<String> result3 = Stream.map(date -> date.user);
        // result1.print();
      //  result2.print();
        result3.print();
        env.execute();
    }
    ///自定义MapFunction
    public static class MyMapper implements MapFunction<Event,String>{
        @Override
        public String map(Event event) throws Exception {
            return event.user;
        }
    }
}
```

###### 4.1.2 过滤（filter）

filter转换操作，顾名思义是对数据流执行一个过滤，通过一个布尔条件表达式设置过滤条件，对于每一个流内元素进行判断，若为true则元素正常输出，若为 false 则元素被过滤掉

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
public class TransformFilterTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L)
        );

        //1.传入一个实现了FilterFunction的类的对象
        SingleOutputStreamOperator<Event> result1 = Stream.filter(new MyFilter());

        //2.传入一个匿名类实现FilterFunction接口
        SingleOutputStreamOperator<Event> result2 = Stream.filter(new FilterFunction<Event>() {
            @Override
            public boolean filter(Event event) throws Exception {
                return event.user.equals("Mary");
            }
        });
        //3.传入Lambda表达式
        Stream.filter(data -> data.user.equals("Alice")).print();
        // result1.print();
       // result2.print();
        env.execute();
    }

    //实现一个自定义的FilterFunction
    public static class MyFilter implements FilterFunction<Event>{
        @Override
        public boolean filter(Event event) throws Exception {
            return event.user.equals("Bob");
        }
    }
}
```

###### 4.1.3 flatMap

 flatMap操作又称为扁平映射，主要是将数据流中的整体（一般是集合类型）拆分成一个一个的个体使用。消费一个元素，可以产生0到多个元素。flatMap可以认为是“扁平化”（flatten） 和“映射”（map）两步操作的结合，也就是先按照某种规则对数据进行打散拆分，再对拆分后的元素做转换处理，之前的WordCount 程序的第一步分词操作，就用到了flatMap

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.typeinfo.TypeHint;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class TransformFlatMapTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L)
        );
        //1.实现FlatMapFunction
       // Stream.flatMap(new MyFlatMap()).print();
        //2.传入一个匿名类实现FlatMapFunction接口
        SingleOutputStreamOperator<String> result1 = (SingleOutputStreamOperator<String>) Stream.flatMap(new FlatMapFunction<Event, String>() {
            @Override
            public void flatMap(Event value, Collector<String> out) throws Exception {
                out.collect(value.user);
                out.collect(value.url);
                out.collect(value.timestamp.toString());
            }
        });
        //3.直接传入Lambda表达式
        SingleOutputStreamOperator<String> result2 = Stream.flatMap((Event value, Collector<String> out) -> {
            if (value.user.equals("Mary"))
                out.collect(value.url);
            else if (value.user.equals("Bob")) {
                out.collect(value.user);
                out.collect(value.url);
                out.collect(value.timestamp.toString());
            }
        }).returns(new TypeHint<String>() {});
        result2.print();
        env.execute();
    }

    //实现一个自定义的FlatMapFunction
    public static class MyFlatMap implements FlatMapFunction<Event,String>{
        @Override
        public void flatMap(Event event, Collector<String> collector) throws Exception {
            collector.collect(event.user);
            collector.collect(event.url);
            collector.collect(event.timestamp.toString());
        }
    }
}
```

###### 4.2 聚合算子

###### 4.2.1 按键分区 keyBy

对于Flink 而言，DataStream是没有直接进行聚合的API的。因为对海量数据做聚合肯定要进行分区并行处理，这样才能提高效率。所以在 Flink 中，要做聚合，需要先进行分区；这个操作就是通过keyBy来完成的;

keyBy是聚合前必须要用到的一个算子。keyBy通过指定键（key），可以将一条流从逻辑上划分成不同的分区（partitions）。这里所说的分区，其实就是并行处理的子任务，也就对应着任务槽（taskslot）。 基于不同的key，流中的数据将被分配到不同的分区中去；这样一来，所有具有相同的key的数据，都将被发往同一个分区，那么下一步算子操作就将会在同一个slot中进行处理了。

在内部，是通过计算 key 的哈希值（hash code），对分区数进行取模运算来实现的。所以这里key如果是POJO的话，必须要重写hashCode()方法。 keyBy()方法需要传入一个参数，这个参数指定了一个或一组key。有很多不同的方法来指定key：比如对于Tuple数据类型，可以指定字段的位置或者多个位置的组合；对于 POJO 类 型，可以指定字段的名称（String）；另外，还可以传入Lambda表达式或者实现一个键选择器 （KeySelector），用于说明从数据中提取key的逻辑。 可以以id作为key 做一个分区操作

需要注意的是，keyBy得到的结果将不再是DataStream，而是会将DataStream转换为KeyedStream。KeyedStream可以认为是“分区流”或者“键控流”，它是对 DataStream 按照 key 的一个逻辑分区，所以泛型有两个类型：除去当前流中的元素类型外，还需要指定key的类型。 KeyedStream也继承自DataStream，所以基于它的操作也都归属于DataStreamAPI。但它跟之前的转换操作得到的SingleOutputStreamOperator不同，只是一个流的分区操作，并不是一个转换算子。KeyedStream是一个非常重要的数据结构，只有基于它才可以做后续的聚合操作（比如sum，reduce）而且它可以将当前算子任务的状态（state）也按照key进行划分、限定为仅对当前key有效

###### 4.2.2 简单聚合

- sum()：在输入流上，对指定的字段做叠加求和的操作。
- min()：在输入流上，对指定的字段求最小值。
- max()：在输入流上，对指定的字段求最大值。
- minBy()：与 min()类似，在输入流上针对指定字段求最小值。不同的是，min()只计 算指定字段的最小值，其他字段会保留最初第一个数据的值；而 minBy()则会返回包 含字段最小值的整条数据。
- maxBy()：与 max()类似，在输入流上针对指定字段求最大值。两者区别与 min()/minBy()完全一致。

**指定位置**

```java
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformTupleAggTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Tuple2<String, Integer>> stream = env.fromElements(
                Tuple2.of("a", 1),
                Tuple2.of("a", 3),
                Tuple2.of("b", 3),
                Tuple2.of("b", 4)
        );
        stream.keyBy(r -> r.f0).sum(1).print();
        stream.keyBy(r -> r.f0).sum("f1").print();
        stream.keyBy(r -> r.f0).max(1).print();
        stream.keyBy(r -> r.f0).max("f1").print();
        stream.keyBy(r -> r.f0).min(1).print();
        stream.keyBy(r -> r.f0).min("f1").print();
        stream.keyBy(r -> r.f0).maxBy(1).print();
        stream.keyBy(r -> r.f0).maxBy("f1").print();
        stream.keyBy(r -> r.f0).minBy(1).print();
        stream.keyBy(r -> r.f0).minBy("f1").print();
        env.execute();
    }
}
```

**指定字段**

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransFormSimpleAggTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L),
                new Event("Bob", "./home", 3300L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)

        );
        //按键分组之后进行聚合,提取当前用户最后一次访问数据
        Stream.keyBy(new KeySelector<Event, String>() {
            @Override
            public String getKey(Event value) throws Exception {
                return value.user;
            }
        }).max("timestamp").print("Max");
        //指定字段的名称
        Stream.keyBy(data -> data.user).maxBy("timestamp").print("MapBy");
        env.execute();
    }
}
```

###### 4.2.3 归约聚合（reduce）

如果说简单聚合是对一些特定统计需求的实现，那么reduce算子就是一个一般化的聚合统计操作了。从MapReduce开始，对reduce操作就不陌生：它可以对已有的数据进行归约处理，把每一个新输入的数据和当前已经归约出来的值，再做一个聚合计算。
与简单聚合类似，reduce操作也会将KeyedStream转换为DataStream。它不会改变流的元素数据类型，所以输出类型和输入类型是一样的。 调用KeyedStream的reduce方法时，需要传入一个参数，实现 ReduceFunction接口。接 口在源码中的定义如下

```java
public interface ReduceFunction<T> extends Function, Serializable {
	T reduce(T value1, T value2) throws Exception;
}
```

ReduceFunction接口里需要实现 reduce()方法，这个方法接收两个输入事件，经过转换处理之后输出一个相同类型的事件；所以，对于一组数据，可以先取两个进行合并，然后再将合并的结果看作一个数据、再跟后面的数据合并，最终会将它“简化”成唯一的一个数据， 这也就是 reduce“归约”的含义。在流处理的底层实现过程中，实际上是将中间“合并的结果” 作为任务的一个状态保存起来的；之后每来一个新的数据，就和之前的聚合状态进一步做归约。

其实，reduce的语义是针对列表进行规约操作，运算规则由ReduceFunction中的reduce方法来定义，而在ReduceFunction内部会维护一个初始值为空的累加器，注意累加器的类型和输入元素的类型相同，当第一条元素到来时，累加器的值更新为第一条元素的值，当新的元素到来时，新元素会和累加器进行累加操作，这里的累加操作就是reduce函数定义的运算规则。然后将更新以后的累加器的值向下游输出。

可以单独定义一个函数类实现ReduceFunction接口，也可以直接传入一个匿名类。 当然，同样也可以通过传入Lambda表达式实现类似的功能。 与简单聚合类似，reduce操作也会将KeyedStream转换为DataStrema。它不会改变流的元素数据类型，所以输出类型和输入类型是一样的。 下面来看一个稍复杂的例子:

将数据流按照用户id进行分区，然后用一个reduce算子实现sum的功能，统计每个用户访问的频次；进而将所有统计结果分到一组，用另一个reduce算子实现 maxBy的功能， 记录所有用户中访问频次最高的那个，也就是当前访问量最大的用户是谁

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
public class TransformReduceTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)

        );
        //1.统计每个用户的访问频次
        SingleOutputStreamOperator<Tuple2<String, Long>> clicksByUser = Stream.map(new MapFunction<Event, Tuple2<String, Long>>() {
            //将Event数据类型转换成元组类型
            @Override
            public Tuple2<String, Long> map(Event value) throws Exception {
                return Tuple2.of(value.user, 1L);
            }
        }).keyBy(data -> data.f0) //使用用户名来进行分流
                .reduce(new ReduceFunction<Tuple2<String, Long>>() {
            @Override
            public Tuple2<String, Long> reduce(Tuple2<String, Long> value1, Tuple2<String, Long> value2) throws Exception {
                //每到一条数据，用户 pv 的统计值加 1
                return Tuple2.of(value1.f0, value1.f1 + value2.f1);
            }
        });
        //2.选取当前最活跃的用户
        SingleOutputStreamOperator<Tuple2<String, Long>> result = clicksByUser
                .keyBy(data -> "key") //为每一条数据分配同一个key，将聚合结果发送到一条流中去
                .reduce(new ReduceFunction<Tuple2<String, Long>>() {
            @Override
            public Tuple2<String, Long> reduce(Tuple2<String, Long> value1, Tuple2<String, Long> value2) throws Exception {
                // 将累加器更新为当前最大的 pv 统计值，然后向下游发送累加器的值
                return value1.f1 > value2.f1 ? value1 : value2;
            }
        });
        result.print();
        env.execute();
    }
}
```

###### 4.2.4 用户自定义函数UDF

* **函数类(FunctionClasses)**

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TransformFilterTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L)
        );

        //1.传入一个实现了FilterFunction的类的对象
        SingleOutputStreamOperator<Event> result1 = Stream.filter(new MyFilter());
        result1.print();
        env.execute();
    }
    //实现一个自定义的FilterFunction
    public static class MyFilter implements FilterFunction<Event>{
        @Override
        public boolean filter(Event event) throws Exception {
            return event.user.equals("Bob");
        }
    }
}
```

```java
//传入一个匿名类实现FilterFunction接口
        SingleOutputStreamOperator<Event> result2 = Stream.filter(new FilterFunction<Event>() {
            @Override
            public boolean filter(Event event) throws Exception {
                return event.user.equals("Mary");
            }
        });
```

```java
 SingleOutputStreamOperator<Event> result1 = Stream.filter(new MyFilter("Bob"));
    //实现一个自定义的FilterFunction
    public static class MyFilter implements FilterFunction<Event>{
        private String UserName;
        MyFilter(String userName) {
            this.UserName = userName;
        }

        @Override
        public boolean filter(Event event) throws Exception {

            return event.user.equals(this.UserName);
        }
    }
```

* **匿名函数（Lambda）**

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
public class TransformLambdaTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Event> clicks = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L)
        );
        //map 函数使用Lambda 表达式，返回简单类型，不需要进行类型声明
        clicks.map(data -> data.url).print();
        env.execute();
    }
}
```

* **富函数类(RichFunctionClasses)**

“富函数类”也是DataStream API提供的一个函数类的接口，所有的Flink函数类都有其Rich版本。富函数类一般是以抽象类的形式出现的。
例如：RichMapFunction、RichFilterFunction、RichReduceFunction 等。
既然“富”，那么它一定会比常规的函数类提供更多、更丰富的功能。与常规函数类的不同主要在于，富函数类可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能。
注：生命周期的概念在编程中其实非常重要，到处都有体现。例如：对于C语言来说，需要手动管理内存的分配和回收，也就是手动管理内存的生命周期。分配内存而不回收，会造成内存泄漏，回收没有分配过的内存，会造成空指针异常。而在JVM 中，虚拟机会自动帮助管理对象的生命周期。对于前端来说，一个页面也会有生命周期。数据库连接、网络连接以及文件描述符的创建和关闭，也都形成了生命周期。所以生命周期的概念在编程中是无处不在的，需要多加注意。
Rich Function有生命周期的概念。典型的生命周期方法有：

- open()方法，是Rich Function 的初始化方法，也就是会开启一个算子的生命周期。当一个算子的实际工作方法例如map()或者filter()方法被调用之前，open()会首先被调用。所以像文件IO 的创建，数据库连接的创建，配置文件的读取等等这样一次性的工作，都适合在open()方法中完成。
- close()方法，是生命周期中的最后一个调用的方法，类似于解构方法。一般用来做一些清理工作。

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
public class TransformRichFunction {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 2000L),
                new Event("Alice", "./prod?id=100", 3000L)
        );
        Stream.map(new MyRichMapper()).setParallelism(2)
                .print();
        env.execute();
    }
    //实现一个自定义的富函数类
    public static class MyRichMapper extends RichMapFunction<Event,Integer>{
        @Override
        public void open(Configuration parameters) throws Exception {
            super.open(parameters);
            System.out.println("Open生命周期被调用 " + getRuntimeContext().getIndexOfThisSubtask()+ "号任务启动");
        }
        @Override
        public Integer map(Event value) throws Exception {
            return value.url.length();
        }
        @Override
        public void close() throws Exception {
            super.close();
            System.out.println("Close生命周期被调用 " + getRuntimeContext().getIndexOfThisSubtask() +Myyang "号任务启结束");
        }
    }
}
```

```java
public class MyFlatMap extends RichFlatMapFunction<IN, OUT>> { 
    @Override
    public void open(Configuration configuration) { 

        // 做一些初始化工作
        // 例如建立一个和MySQL 的连接
    }
    @Override
    public void flatMap(IN in, Collector<OUT out) {
        // 对数据库进行读写

    }
    @Override
    public void close() {
        // 清理工作，关闭和MySQL 数据库的连接。

    }
}
```

##### 5 物理分区

顾名思义，“分区”（partitioning）操作就是要将数据进行重新分布，传递到不同的流分区去进行下一步处理。其实应该对分区操作并不陌生，前面介绍聚合算子时，已经提到了keyBy，它就是一种按照键的哈希值来进行重新分区的操作。只不过这种分区操作只能保证把数据按key“分开”，至于分得均不均匀、每个key 的数据具体会分到哪一区去，这些是完全无从控制的——所以有时也说keyBy是一种逻辑分区（logical partitioning）操作

如果说keyBy这种逻辑分区是一种“软分区”，那真正硬核的分区就应该是所谓的“物理分区”（physical partitioning）。也就是要真正控制分区策略，精准地调配数据，告诉每个数据到底去哪里。其实这种分区方式在一些情况下已经在发生了：例如编写的程序可能对多个处理任务设置了不同的并行度，那么当数据执行的上下游任务并行度变化时，数据就不应该还在当前分区以直通（forward）方式传输了——因为如果并行度变小，当前分区可能没有下游任务了；而如果并行度变大，所有数据还在原先的分区处理就会导致资源的浪费。所以这种情况下，系统会自动地将数据均匀地发往下游所有的并行任务，保证各个分区的负载均衡

有些时候，还需要手动控制数据分区分配策略。比如当发生数据倾斜的时候，系统无法自动调整，这时就需要重新进行负载均衡，将数据流较为平均地发送到下游任务操作分区中去。Flink 对于经过转换操作之后的DataStream，提供了一系列的底层操作接口，能够帮实现数据流的手动重分区。为了同keyBy相区别，把这些操作统称为“物理分区”操作。物理分区与keyBy另一大区别在于，keyBy之后得到的是一个KeyedStream，而物理分区之后结果仍是DataStream，且流中元素数据类型保持不变。从这一点也可以看出，分区算子并不对数据进行转换处理，只是定义了数据的传输方式。
常见的物理分区策略有随机分配（Random）、轮询分配（Round-Robin）、重缩放（Rescale）和广播（Broadcast），下边分别来做了解

1. **随机分区（shuffle）**

最简单的重分区方式就是直接“洗牌”。通过调用DataStream的.shuffle()方法，将数据随机地分配到下游算子的并行任务中去。
随机分区服从均匀分布（uniform distribution），所以可以把流中的数据随机打乱，均匀地传递到下游任务分区，因为是完全随机的，所以对于同样的输入数据, 每次执行得到的结果也不会相同
经过随机分区之后，得到的依然是一个DataStream
可以做个简单测试：将数据读入之后直接打印到控制台，将输出的并行度设置为4，中间经历一次shuffle。执行多次，观察结果是否相同

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.Partitioner;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;
public class TransformPartitionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)

        );
        //1、随机分区
        Stream.shuffle().print().setParallelism(4);
        env.execute();
    }
}
```

2. **轮询分区（Round-Robin）**

轮询也是一种常见的重分区方式。简单来说就是“发牌”，按照先后顺序将数据做依次分发。通过调用DataStream的.rebalance()方法，就可以实现轮询重分区。rebalance使用的是Round-Robin负载均衡算法，可以将输入流数据平均分配到下游的并行任务中去。
注：**Round-Robin算法用在了很多地方，例如Kafka 和Nginx**

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.Partitioner;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;
public class TransformPartitionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)
        );
        //2、轮询分区
       Stream.rebalance().print().setParallelism(4);
       Stream.print().setParallelism(4);  //输出和rebalance一致。Flink底层默认就是 rebalance 分区
        env.execute();
    }
}
```

3. **重缩放分区（rescale）**

重缩放分区和轮询分区非常相似。当调用rescale()方法时，其实底层也是使用Round-Robin算法进行轮询，但是只会将数据轮询发送到下游并行任务的一部分中，也就是说，“发牌人”如果有多个，那么rebalance的方式是每个发牌人都面向所有人发牌；而rescale的做法是分成小团体，发牌人只给自己团体内的所有人轮流发牌。

当下游任务（数据接收方）的数量是上游任务（数据发送方）数量的整数倍时，rescale的效率明显会更高。比如当上游任务数量是2，下游任务数量是6 时，上游任务其中一个分区的数据就将会平均分配到下游任务的3 个分区中。

由于rebalance是所有分区数据的“重新平衡”，当TaskManager数据量较多时，这种跨节点的网络传输必然影响效率；而如果配置的taskslot数量合适，用rescale的方式进行“局部重缩放”，就可以让数据只在当前TaskManager的多个slot之间重新分配，从而避免了网络传输带来的损耗。
从底层实现上看，rebalance和rescale的根本区别在于任务之间的连接机制不同。rebalance将会针对所有上游任务（发送数据方）和所有下游任务（接收数据方）之间建立通信通道，这是一个笛卡尔积的关系；而 rescale 仅仅针对每一个任务和下游对应的部分任务之间建立通信通道，节省了很多资源。

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.Partitioner;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;
public class TransformPartitionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)

        );
        //3、rescale重缩放分区
        //这里使用了并行数据源的富函数版本
        //这样可以调用getRuntimeContext方法来获取运行时上下文的一些信息 
        env.addSource(new RichParallelSourceFunction<Integer>() {
            @Override
            public void run(SourceContext<Integer> ctx) throws Exception {
                for (int i = 0; i < 8; i++) {
                    //将奇偶数发送到0号和1号并行分区
                    if(i % 2 == getRuntimeContext().getIndexOfThisSubtask())
                        ctx.collect(i);
                }
            }

            @Override
            public void cancel() {
            }
        }).setParallelism(2);
     //   .rescale().print().setParallelism(4);
        env.execute();
    }
}
```

4.  **广播（broadcast）**

这种方式其实不应该叫做“重分区”，因为经过广播之后，数据会在不同的分区都保留一份，可能进行重复处理。可以通过调用DataStream的broadcast()方法，将输入数据复制并发送到下游算子的所有并行任务中去

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.Partitioner;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;
public class TransformPartitionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)
        );
        //4、广播
        Stream.broadcast().print().setParallelism(4);
        env.execute();
    }
}
```

5. **全局分区（global）**

全局分区也是一种特殊的分区方式。这种做法非常极端，通过调用.global()方法，会将所有的输入流数据都发送到下游算子的第一个并行子任务中去。这就相当于强行让下游任务并行度变成了1，所以使用这个操作需要非常谨慎，可能对程序造成很大的压力

```java
Stream.global().print().setParallelism(4);
```

6.  **自定义分区（Custom）**

当Flink提供的所有分区策略都不能满足用户的需求时，可以通过使用partitionCustom()方法来自定义分区策略。
在调用时，方法需要传入两个参数，第一个是自定义分区器（Partitioner）对象，第二个是应用分区器的字段，它的指定方式与keyBy指定 key 基本一样：可以通过字段名称指定，也可以通过字段位置索引来指定，还可以实现一个KeySelector。

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.Partitioner;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.ParallelSourceFunction;
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;
public class TransformPartitionTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //从元素中读取数据
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)

        );
        //6、自定义分区
        //将自然数按照奇偶分区
        env.fromElements(1,2,3,4,5,6,7,8).partitionCustom(new Partitioner<Integer>() {
            @Override
            public int partition(Integer key, int numPartitions) {
                return key % 2;
            }
        }, new KeySelector<Integer, Integer>() {
            @Override
            public Integer getKey(Integer value) throws Exception {
                return value;
            }
        }).print().setParallelism(4);
        env.execute();
    }
}
```

##### 6 Sink

在Flink中，如果希望将数据写入外部系统，其实并不是一件难事。所有算子都可以通过实现函数类来自定义处理逻辑，所以只要有读写客户端，与外部系统的交互在任何一个处理算子中都可以实现。例如在MapFunction中，完全可以构建一个到Redis的连接，然后将当前处理的结果保存到Redis中。如果考虑到只需建立一次连接，也可以利用RichMapFunction，在open()生命周期中做连接操作。这样看起来很方便，却会带来很多问题。Flink作为一个快速的分布式实时流处理系统，对稳定性和容错性要求极高。一旦出现故障，应该有能力恢复之前的状态，保障处理结果的正确性。这种性质一般被称作“状态一致性”。Flink内部提供了一致性检查点（checkpoint）来保障可以回滚到正确的状态；但如果在处理过程中任意读写外部系统，发生故障后就很难回退到从前了。为了避免这样的问题，Flink的DataStreamAPI专门提供了向外部写入数据的方法：**addSink**

```java
 public DataStreamSink<T> print() {
        PrintSinkFunction<T> printFunction = new PrintSinkFunction<>();
        return addSink(printFunction).name("Print to Std. Out");
    }
```

```java
stream.addSink(new SinkFunction(…));\
```

```java
default void invoke(IN value, Context context) throws Exception
```

1. 输出到文件

   最简单的输出方式，当然就是写入文件了。对应读取文件作为输入数据源，Flink本来也有一些非常简单粗暴的输出到文件的预实现方法：如writeAsText()、writeAsCsv()，可以直接将输出结果保存到文本文件或Csv文件。但是，这种方式是不支持同时写入一份文件的；所以往往会将最后的Sink操作并行度设为1，这就大大拖慢了系统效率；而且对于故障恢复后的状态一致性，也没有任何保证。所以目前这些简单的方法已经要被弃用。 Flink为此专门提供了一个流式文件系统的连接器：**StreamingFileSink**，它继承自抽象类**RichSinkFunction**，而且集成了Flink的检查点（checkpoint）机制，用来保证精确一次（exactly once）的一致性语义。 StreamingFileSink为批处理和流处理提供了一个统一的Sink，它可以将分区文件写入Flink支持的文件系统。它可以保证精确一次的状态一致性，大大改进了之前流式文件 Sink 的方式。 它的主要操作是将数据写入桶（buckets），每个桶中的数据都可以分割成一个个大小有限的分区文件，这样一来就实现真正意义上的分布式文件存储。可以通过各种配置来控制“分桶” 的操作；默认的分桶方式是基于时间的，每小时写入一个新的桶。换句话说，每个桶内保存的文件，记录的都是1小时的输出数据。

   

StreamingFileSink支持行编码（Row-encoded）和批量编码（Bulk-encoded，比如Parquet） 格式。这两种不同的方式都有各自的构建器（builder），调用方法也非常简单，可以直接调用 StreamingFileSink 的静态方法

- 行编码：StreamingFileSink.forRowFormat（basePath，rowEncoder）。
- 批量编码：StreamingFileSink.forBulkFormat（basePath，bulkWriterFactory）。

在创建行或批量编码Sink 时，需要传入两个参数，用来指定存储桶的基本路径（basePath）和数据的编码逻辑（rowEncoder或bulkWriterFactory）

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.serialization.SimpleStringEncoder;
import org.apache.flink.core.fs.Path;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink;
import org.apache.flink.streaming.api.functions.sink.filesystem.rollingpolicies.DefaultRollingPolicy;
import java.util.concurrent.TimeUnit;
public class SinkToFileTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);

        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)
        );
        StreamingFileSink<String> stringStreamingFileSink = StreamingFileSink.<String>forRowFormat(new Path("./OutPut"),
                new SimpleStringEncoder<>("UTF-8"))
                .withRollingPolicy(
                      DefaultRollingPolicy.builder()
                              .withMaxPartSize(1024 * 1024 * 1024)
                              .withRolloverInterval(TimeUnit.MINUTES.toMillis(15))
                              .withInactivityInterval(TimeUnit.MINUTES.toMillis(5))
                              .build()
                )
                .build();

        Stream.map(data -> data.toString())
                .addSink(stringStreamingFileSink);
        env.execute();
    }
}
```

这里创建了一个简单的文件Sink，通过.withRollingPolicy()方法指定了一个“滚动策略”。
“滚动”的概念在日志文件的写入中经常遇到：因为文件会有内容持续不断地写入，所以应该给一个标准，到什么时候就开启新的文件，将之前的内容归档保存。也就是说，上面的代码设置了在以下3种情况下，就会滚动分区文件：

- 至少包含15 分钟的数据;
- 最近 5 分钟没有收到新的数据;
- 文件大小已达到1GB;

2. kafka

Kafka是一个分布式的基于发布/订阅的消息系统，本身处理的也是流式数据，所以和Flink是“天生一对”，经常会作为Flink的输入数据源和输出系统。Flink官方为Kafka提供了Source和Sink的连接器，可以用它方便地从Kafka 读写数据

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;

import java.util.Properties;

public class SinkToKafka {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        //1.从Kafka读取数据
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "hadoop102:9092");
        DataStreamSource<String> kafkaStream = env.addSource(new FlinkKafkaConsumer<String>("clicks", new SimpleStringSchema(), properties));

        //2,用Flink进行转换处理

        SingleOutputStreamOperator<String> result = kafkaStream.map(new MapFunction<String, String>() {
            @Override
            public String map(String value) throws Exception {
                String[] fields = value.split(",");
                return new Event(fields[0].trim(), fields[1].trim(), Long.valueOf(fields[2].trim())).toString();
            }
        });

        //3,结果数据写入到Kafka
        result.addSink(new FlinkKafkaProducer<String>("hadoop102:9092","Events",new SimpleStringSchema()));

        // 1> 启动Zk           三个机器都要启动 都要运行 $ zkServer.sh start  查看状态: zkServer.sh status
        // 2> 启动Kafka：      三个机器都要启动  都要运行 $ kafka-server-start.sh /opt/module/kafka_2.13-3.2.0/config/server.properties &
        // 3> 创建生产者:                             $ kafka-console-producer.sh --broker-list hadoop102:9092 --topic clicks
        // 4> 创建消费者:                             $ kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --topic Events
        // 5> 运行代码  在生产者端复制数据  在消费者端查看消费的数据
        // zkServer.sh status
        env.execute();
    }
}
```

这里可以看到，addSink传入的参数是一个FlinkKafkaProducer。这也很好理解，因为需要向Kafka写入数据，自然应该创建一个生产者。FlinkKafkaProducer 继承了抽象类 TwoPhaseCommitSinkFunction，这是一个实现了“两阶段提交”的 RichSinkFunction。两阶段提交提供了Flink向Kafka写入数据的事务性保证，能够真正做到精确一次（exactly once）的状态一致性

3. redis 

Redis是一个开源的内存式的数据存储，提供了像字符串（string）、哈希表（hash）、列表（list）、集合（set）、排序集合（sorted set）、位图（bitmap）、地理索引和流（stream）等一系列常用的数据结构。因为它运行速度快、支持的数据类型丰富，在实际项目中已经成为了架构优化必不可少的一员，一般用作数据库、缓存，也可以作为消息代理。 Flink没有直接提供官方的Redis连接器，不过Bahir项目还是担任了合格的辅助角色，提供了Flink-Redis的连接工具。但因版本升级略显滞后，目前连接器版本为1.0，支持的Scala版本最新到2.11。由于测试不涉及到Scala的相关版本变化，所以并不影响使用。 在实际项目应用中，应该以匹配的组件版本运行

```java
import com.kunan.StreamAPI.Source.ClickSource;
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.redis.RedisSink;
import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommand;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisCommandDescription;
import org.apache.flink.streaming.connectors.redis.common.mapper.RedisMapper;
public class SinkToRedis {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Event> stream = env.addSource(new ClickSource());
        // 创建一个jedis连接配置
        FlinkJedisPoolConfig config = new FlinkJedisPoolConfig.Builder()
                .setHost("hadoop102")
                .setPort(6379)
                .setTimeout(30000)
                .build();
        //写入Redis
        //JFlinkJedisConfigBase：Jedis 的连接配置
		//RedisMapper：Redis 映射类接口，说明怎样将数据转换成可以写入 Redis 的类型
        stream.addSink(new RedisSink<Event>(config,new MyRedisMapper()));
        env.execute();
    }
    //自定义类实现RedisMapper接口
    public static class MyRedisMapper implements RedisMapper<Event>{
        @Override
        public RedisCommandDescription getCommandDescription() {
            return new RedisCommandDescription(RedisCommand.HSET,"clicks");
        }
        @Override
        public String getKeyFromData(Event data) {
            return data.user;
        }
        @Override
        public String getValueFromData(Event data) {
            return data.url;
        }
    }
}
```

4. es

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.api.common.functions.RuntimeContext;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.elasticsearch.ElasticsearchSinkFunction;
import org.apache.flink.streaming.connectors.elasticsearch.RequestIndexer;
import org.apache.flink.streaming.connectors.elasticsearch7.ElasticsearchSink;
import org.apache.http.HttpHost;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.client.Requests;

import java.util.ArrayList;
import java.util.HashMap;

public class SinkToES {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)
        );
        //定义hots列表
        ArrayList<HttpHost> httpHosts = new ArrayList<>();
        httpHosts.add(new HttpHost("hadoop102",9200));

        //定义ElasticsearchFunction
        ElasticsearchSinkFunction<Event> elasticsearchSinkFunction = new ElasticsearchSinkFunction<Event>() {
            @Override
            public void process(Event event, RuntimeContext runtimeContext, RequestIndexer requestIndexer) {
                HashMap<String, String> map = new HashMap<>();
                map.put(event.user, event.url);

                //构建一个IndexRequest
                IndexRequest request = Requests.indexRequest()
                        .index("clicks")
                        .source(map);
                requestIndexer.add(request);
            }
        };

        //写入ES
        Stream.addSink(new ElasticsearchSink.Builder<>(httpHosts,elasticsearchSinkFunction).build());
        env.execute();
    }
}
```

5 mysql

```java
import com.kunan.StreamAPI.Source.Event;
import org.apache.flink.connector.jdbc.JdbcConnectionOptions;
import org.apache.flink.connector.jdbc.JdbcSink;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
public class SinkToMySQL {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Event> Stream = env.fromElements(
                new Event("Mary", "./home", 1000L),
                new Event("Bob", "./cart", 1500L),
                new Event("Alice", "./prod?id=100", 1800L),
                new Event("Bob", "./prod?id=1", 2000L),
                new Event("Alice", "./prod?id=200", 3000L),
                new Event("Bob", "./home", 2500L),
                new Event("Bob", "./prod?id=120", 3600L),
                new Event("Bob", "./prod?id=130", 4000L)
        );

        Stream.addSink(JdbcSink.sink(
                "INSERT INTO CLICKS(USER,URL) VALUES (? , ?)",
                ((statement,event) -> {
                    statement.setString(1,event.user);
                    statement.setString(2,event.url);
                }),
                new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                        .withUrl("jdbc:mysql://hadoop102:3306/My_Flink_Test")
                        .withDriverName("com.mysql.jdbc.Driver")
                        .withUsername("root")
                        .withPassword("000000")
                        .build()
        ));
        env.execute();
    }
}
```