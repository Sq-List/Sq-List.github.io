---
title: Flink笔记
date: 2021-03-02 22:59:24
tags:
	- flink
categories:
	- flink
---

## 1 Flink 运行架构

### 1.1 Flink运行时的组件

Flink运行时架构主要包括四个不同的组件：

* 作业管理器（JobManager）
* 资源管理器（ResourceManager）
* 任务管理器（TaskManager）
* 分发器（Dispatcher）

<!-- mode -->

#### 作业管理器（JobManager）

控制一个应用程序执行的主进程，每一个应用程序都会被一个**不同**的JobManager所控制执行。

**（集群配置时候的JobManager只是指JobManager的节点，用来启动JobManager组件）**

JobManager会先接收到要执行的应用程序，这个应用程序包括：

* 作业图（JobGraph）
* 逻辑数据流图（logical dataflow graph）
* 打包了所有的类、库和其他资源的JAR包

职能：

* 将JobGraph转换成一个物理层面的数据流图，“执行图”（ExecutionGraph），包含了所有可以并发执行的任务；
* 向资源管理器（ResourceManager）请求执行任务必要的资源，即任务管理器上（TaskManager）上的插槽（slot），将执行图分发到执行任务的TaskManager上；
* 负责所有需要中央协调的操作，比如检查点（checkpoints）的协调。

#### 资源管理器（ResourceManager）

主要负责管理任务管理（TaskManager）的插槽（slot），slot是Flink中定义的处理资源单元。

Flink为不同环境和资源管理工具提供了不同的资源管理器，比如：YARN、Mesos、K8s以及Standalone。

JobManager申请slot资源时，ResourceManager会将有空闲slot的TaskManager分配给JobManager。

ResourceManager没有足够的slot来满足JobManager请求时，会向资源提供平台发起会话，以提供启动TaskManager集成的容器。

#### 任务管理器（TaskManager）

每一个TaskManager都包含了一定数量的slot。slot限制了TaskManager能够执行的任务数量。

一个TaskManager可以跟其他运行同一程序的TaskManager交换数据。

#### 分发器（Dispatcher）

可以跨作业运行，为应用提交提供了REST接口。

当一个应用被提交执行时，Dispatcher会启动并将应用移交给一个JobManager。

Dispatcher也会启动一个Web UI，方便地展示和监控作业执行信息。



### 1.2 任务提交流程

![任务提交和组件交互流程](https://ww1.sinaimg.cn/large/6af0fe46ly1go6z60iq4lj20is04y74e.jpg)

图上是一个较为高层级的视角。

如果将Flink集群部署到YARN上，那么就有一下提交流程：

![Yarn模式任务提交流程](https://ww1.sinaimg.cn/large/6af0fe46ly1go7vuybqa0j20lm09dmxf.jpg)

1. Flink任务提交后，Client向HDFS上传Flink的jar包和配置；
2. Client向ResourceManager（Yarn）提交任务，ResourceManager（Yarn）分配Container资源并通知对应的NodeManager启动ApplicationMaster；
3. ApplicationMaster启动后加载Flink的jar包，启动ResourceManager（Flink）和JobManager；
4. JobManager向ResourceManager（Flink）申请资源，ResourceManager（Flink）向ResourceManager（Yarn）申请资源启动TaskManager；
5. ResourceManager（Yarn）分配Container资源后，由ApplicationMaster通知资源所在节点的NodeManager启动TaskManager；
6. NodeManager 加载 Flink 的 Jar 包和配置构建环境并启动 TaskManager；
7. TaskManager 启动后向 JobManager 发送心跳包，并等待 JobManager 向其分配任务。



### 1.3 任务调度原理

![任务调度原理](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnflrcbb7j20i70cbaf3.jpg)

* 客户端不是程序执行的一部分，但它用于准备并发送dataflow（JobGraph）给Master（JobManager）；然后客户端断开连接或者维持连接以等待接收计算结果；
* Flink集群启动后，会先启动一个JobManager和一个或多个的TaskManager。
  * 由Client提交任务给JobManager，JobManager再调度任务到各个TaskManager去执行，TaskManager将心跳和统计信息汇报给JobManager。
  * TaskManager之间以流的形式进行数据传输。
  * 以上三者均为独立的JVM进程。
* **Client**为提交Job的客户端，可以是与JobManager环境连通的任何机器。提交Job后，Client可以结束进程，也可以不结束并等待结果返回。
* **JobManager**主要负责调度Job并协调Task做checkPoint。接收到Job和JAR包资源后，会生成优化后的执行计划，并以Task的单元调度到各个TaskManager去执行。
* **TaskManager**在启动时就已经设置好了slot，每个slot能启动一个Task，Task为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自 己的上游建立 Netty 连接，接收数据并处理。

#### 1.3.1 TaskManager与Slots

Flink中每一个worker（TaskManager）都是一个**JVM进程**，它可能会在**独立的线程**上执行一个或多个subtask。

为了控制一个worker能够接受多少个task，worker通过slot来进行控制。

每个slot表示TaskManager拥有资源的**一个固定大小的子集**。假如一个TaskManager有三个slot，那么它会将其管理的内存分成三份给各个slot。资源slot意味着一个subtask不需要跟来自其他job的subtask竞争被管理的内存。（slot目前仅仅用于隔离task的受管理的内存，不涉及CPU的隔离。）

![在这里插入图片描述](https://ww1.sinaimg.cn/large/6af0fe46ly1gocugauxpsj20kw06qdfu.jpg)

**上图这个每个子任务各自占用一个slot，可以在代码中通过算子的`.slotSharingGroup("组名")`指定算子所在的Slot组名，默认每一个算子的SlotGroup和上一个算子相同，而默认的SlotGroup就是"default"**。

通过调整task slot的数量，允许用户定义subtask之间如何相互隔离。如果一个TaskManager一个slot，那意味着每个task group运行在独立的JVM中；而一个TaskManager多个slot意味着更多的subtask可以共享同一个JVM。**而在同一个JVM进程中的task将共享TCP连接和心跳消息，也可能共享数据集和数据结构。**

![subtask共享slot](https://ww1.sinaimg.cn/large/6af0fe46ly1gocu55be2kj20k4097t93.jpg)

默认情况下，Flink允许子任务共享slot，即使他们是不同任务的子任务（它们来自同一个job）。

**Task slot是静态的概念，是指TaskManager具有的并发执行能力，指的是能力！**可以通过flink-conf.yaml中参数taskmanager.numberOfTaskSlots进行配置；而**并行度parallelism是动态概念，即TaskManager运行程序时实际使用的并发能力**，可以通过flink-conf.yaml参数parallelism.default或命令行-p或代码中setParallelism方法进行配置。

![在这里插入图片描述](https://ww1.sinaimg.cn/large/6af0fe46ly1gocud3rworj20ia0seta1.jpg)

#### 1.3.2 程序与数据流（DataFlow）

![在这里插入图片描述](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnfm1emnwj20ha0bo779.jpg)

所有的Flink程序都是由三部分组成的：**Source**、**Transformation**和**Sink**。

* Source：负责读取数据源；
* Transformation：利用各种算子进行处理加工；
* Sink：负责输出。

运行时，Flink上运行的程序会被映射成“逻辑数据流”（dataflows）。

**每一个dataflow以一个或多个Sources开始以一个或多个Sinks结束。**dataflow类似于任意的有向无环图（DAG）。

大部分情况下，程序中的转换运算（transformations）跟dataflow中的算子（operator）是一一对应的关系，但有时候，一个transformation可能对用多个operator。

#### 1.3.3 执行图（ExecutionGraph）

由Flink程序直接映射成的数据流图是StreamGraph，也被称为**逻辑流图**。

为了执行一个流处理程序，Flink需要将**逻辑流图**转换成**物理数据流图（也叫执行图）**。

Flink中的执行图可以分成四层：

```
StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图
```

* StreamGraph：根据用户通过Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。

* JobGraph：StreamGraph经过优化后生成了JobGraph，提交给JobManager的数据结构。主要优化为，将多个符合条件的节点chain在一起做一个节点，这样可以减少数据在节点之间流动所需要的序列化、反序列化、传输消耗。

* ExecutionGraph：JobManager根据JobGraph生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。

* 物理执行图：JobManager根据ExecutionGraph对Job进行调度后，在各个TaskManager上部署Task后形成的“图”，并不是一个具体的数据结构。

  ![在这里插入图片描述](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnfm5ym5fj20h80gttil.jpg)

#### 1.3.4 数据传输形式

一个程序中，不同的算子可能具有不同的并行度。

算子之间传输数据的形式可以是one-to-one(forwarding)的模式也可以是redistributing的模式，具体是哪一种形式，取决于算子的种类。

* One-To-One：stream维护着分区以及元素的顺序（比如Source和Map之间）。这意味着map算子的子任务看到的元素的个数以及顺序跟source算子的子任务生产的元素的个数、顺序相同。**map、fliter、flatMap等算子都是One-To-One的对应关系**；
* Redistributing：stream的分区会发生变化。每一个算子的子任务依据所选择的transformation发送数据到不同的目标任务。**keyBy基于hashCode重分区、而broadcast和rebalance会随机进行重分区，这些算子都会引起redistribute过程，而 redistribute 过程就类似于 Spark 中的 shuffle 过程。**

#### 1.3.5 任务链（OperatorChains）

Flink采用了一种任务链的优化技术，可以在特定条件下减少本地通信的开销。为了满足任务链的要求，必须将两个或多个算子设为**相同的并行度**，并同构本地转发（localforward）的方式进行连接

* **相同并行度的one-to-one**操作，Flink这样相连的算子链连接在一起形成一个task，原来的算子称为里面的subtask；
  * **并行度相同、并且是one-to-one操作，两个条件缺一不可。**

![在这里插入图片描述](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnfma5edij20h80e5ajw.jpg)

**如果前后任务逻辑上可以是OneToOne，且并行度一致，那么就能合并在一个Slot里**（并行度原本是多少就是多少，两者并行度一致）执行。

- keyBy需要根据Hash值分配给不同slot执行，所以只能Hash，不能OneToOne。
- 逻辑上可OneToOne但是并行度不同，那么就会Rebalance，轮询形式分配给下一个任务的多个slot。

PS：

* **代码中如果`算子.disableChaining()`，能够强制当前算子的子任务不参与任务链的合并，即不和其他slot资源合并，但是仍然保留“slot共享”的特性。**
* **如果`StreamExecutionEnvironment env.disableOperatorChaining()`则当前执行环境全局设置算子不参与"任务链的合并"。**
* **如果`算子.startNewChain()`表示不管前面任务链合并与否，从当前算子往后重新计算任务链的合并。通常用于前面强制不要任务链合并，而当前往后又需要任务链合并的特殊场景。**
* *如果`算子.shuffle()`，能够强制算子之后重分区到不同slot执行下一个算子操作，逻辑上也实现了任务不参与任务链合并=>但是仅为“不参与任务链的合并”，这个明显不是最优解操作*



## 2 Flink流处理API

![img](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnfme38lwj20fe035q3j.jpg)

### 2.1 Environment

#### 2.1.1 getExecutionEnvironment

创建一个执行环境，表示当前执行程序的上下文。

* 如果程序时独立调用的，则此方法返回本地执行环境；
* 如果从命令行客户端调用程序以提交到集群，则此方法返回集群的执行环境。

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment(); 

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); 
```

#### 2.1.2 createLocalEnvironment

返回本地执行环境；

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(1);
```

#### 2.1.3 createRemoteEnvironment

返回集群执行环境，将Jar提交到远程服务器。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(1);
```

### 2.2 Transform

#### 2.2.1 算子转换

 在Flink中，**Transformation算子就是将一个或多个DataStream转换为新的DataStream**，可以将多个转换组合成复杂的数据流拓扑。 如下图所示，DataStream会由不同的Transformation操作，转换、过滤、聚合成其他不同的流，从而完成我们的业务要求。

![image-20210720165721288](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnipnxmppj20o00dxgpj.jpg)

### 2.3 支持的数据类型

Flink流应用程序处理的是以数据对象表示的事件流。所以在Flink内部，我们需要能够处理这些对象。它们**需要被序列化和反序列化**，以便通过网络传送它们；或者从状态后端、检查点和保存点读取它们。为了有效地做到这一点，Flink需要明确知道应用程序所处理的数据类型。Flink使用类型信息的概念来表示数据类型，并为每个数据类型生成特定的序列化器、反序列化器和比较器。

 Flink还具有一个类型提取系统，该系统分析函数的输入和返回类型，以自动获取类型信息，从而获得序列化器和反序列化器。但是，在某些情况下，例如lambda函数或泛型类型，需要显式地提供类型信息，才能使应用程序正常工作或提高其性能。

 Flink支持Java和Scala中所有常见数据类型。使用最广泛的类型有以下几种。

#### 2.3.1 基础数据类型

Flink支持所有的Java和Scala基础数据类型，Int, Double, Long, String, …

```java
DataStream<Integer> numberStream = env.fromElements(1, 2, 3, 4);
numberStream.map(data -> data * 2);
```

#### 2.3.2 Java 和 Scala 元组（Tuples）

java不像Scala天生支持元组Tuple类型，java的元组类型由Flink的包提供，默认提供Tuple0~Tuple25

```java
DataStream<Tuple2<String, Integer>> personStream = env.fromElements( 
  new Tuple2("Adam", 17), 
  new Tuple2("Sarah", 23) 
); 
personStream.filter(p -> p.f1 > 18);
```

#### 2.3.3 Java简单对象（JOPO）

java的POJO这里要求必须提供无参构造函数

- 成员变量要求都是public（或者private但是提供get、set方法）

```java
public class Person{
  public String name;
  public int age;
  public Person() {}
  public Person( String name , int age) {
    this.name = name;
    this.age = age;
  }
}
DataStream Pe rson > persons = env.fromElements(
  new Person (" Alex", 42),
  new Person (" Wendy",23)
);
```

#### 2.3.4 其他（Arrays, Lists, Maps, Enums等等）

Flink对Java和Scala中的一些特殊目的的类型也都是支持的，比如Java的ArrayList，HashMap，Enum等等。

### 2.4 实现UDF函数——更细粒度的控制流

#### 2.4.1 函数类（Function Classes）

Flink暴露了所有UDF函数的接口(实现方式为接口或者抽象类)。例如MapFunction, FilterFunction, ProcessFunction等等。

 下面例子实现了FilterFunction接口：

```java
DataStream<String> flinkTweets = tweets.filter(new FlinkFilter()); 
public static class FlinkFilter implements FilterFunction<String> { 
  @Override public boolean filter(String value) throws Exception { 
    return value.contains("flink");
  }
}
```

 还可以将函数实现成匿名类

```java
DataStream<String> flinkTweets = tweets.filter(
  new FilterFunction<String>() { 
    @Override public boolean filter(String value) throws Exception { 
      return value.contains("flink"); 
    }
  }
);
```

 我们filter的字符串"flink"还可以当作参数传进去。

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE "); 
DataStream<String> flinkTweets = tweets.filter(new KeyWordFilter("flink")); 
public static class KeyWordFilter implements FilterFunction<String> { 
  private String keyWord;

  KeyWordFilter(String keyWord) { 
    this.keyWord = keyWord; 
  } 

  @Override public boolean filter(String value) throws Exception { 
    return value.contains(this.keyWord); 
  } 
}
```

#### 2.4.2 富函数（Rich Functions）

“富函数”是DataStream API提供的一个函数类的接口，所有Flink函数类都有其Rich版本。

 **它与常规函数的不同在于，可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能**。

- RichMapFunction
- RichFlatMapFunction
- RichFilterFunction
- …

 Rich Function有一个**生命周期**的概念。典型的生命周期方法有：

- **`open()`方法是rich function的初始化方法，当一个算子例如map或者filter被调用之前`open()`会被调用。**
- **`close()`方法是生命周期中的最后一个调用的方法，做一些清理工作。**
- **`getRuntimeContext()`方法提供了函数的RuntimeContext的一些信息，例如函数执行的并行度，任务的名字，以及state状态**

```java
public static class MyMapFunction extends RichMapFunction<SensorReading, Tuple2<Integer, String>> { 

  @Override public Tuple2<Integer, String> map(SensorReading value) throws Exception {
    return new Tuple2<>(getRuntimeContext().getIndexOfThisSubtask(), value.getId()); 
  } 

  @Override public void open(Configuration parameters) throws Exception { 
    System.out.println("my map open"); // 以下可以做一些初始化工作，例如建立一个和HDFS的连接 
  } 

  @Override public void close() throws Exception { 
    System.out.println("my map close"); // 以下做一些清理工作，例如断开和HDFS的连接 
  } 
}
```

##### 测试代码

```java
com.sqlist.apitest.transform.TransformTest4RichFunction
```

### 2.5 数据重分区操作

重分区操作，在DataStream类中可以看到很多`Partitioner`字眼的类。

**其中`partitionCustom(...)`方法用于自定义重分区**。

##### 测试代码

```java
com.sqlist.apitest.transform.TransformTest5Partition
```



## 3 Flink中的Window

### 3.1 Window

#### 3.1.1 概述

![image-20210415201453850](https://ww1.sinaimg.cn/large/6af0fe46ly1gpkoztraolj20kf04wt92.jpg)

streaming流式计算是一种被涉及用于处理无限数据集的数据处理引擎，而无限数据集是指一种不断增长的本质上无限的数据集，而**window是一种切割无限数据为有限块进行处理的手段**。

**window是无限数据流处理的核心，window将一个无限的stream拆分成有限大小的“bucket”桶，我们可以在这些桶上做计算操作**。

🌰：假设按照时间段划分桶，接收的数据马上能判断放到哪个桶，且多个桶的数据能够并行处理。（迟到的数据也可判断是原本属于哪个桶）

#### 3.1.2 window类型

* 时间窗口（Times Window）
  * 滚动时间窗口
  * 滑动时间窗口
  * 会话窗口
* 计数窗口（Count Window）
  * 滚动计数窗口
  * 滑动计数窗口

Time Window：按照时间生成Window

Count Window：按照指定的数据条数生成一个Window，与时间无关

##### 滚动窗口（Tumbling Windows）

![image-20210419184210807](https://ww1.sinaimg.cn/large/6af0fe46ly1gpp9cxvtpej20io0afweo.jpg)

* 依据**固定的窗口长度**对数据进行切分
* 时间对齐，窗口长度固定，没有重叠

##### 滑动窗口（Sliding Windows）

![image-20210419190744946](https://ww1.sinaimg.cn/large/6af0fe46ly1gpp9gzfiybj20go09r3z8.jpg)

* 可以按照固定的长度向后滑动固定的距离
* 滑动窗口由**固定的窗口长度**和**滑动间隔**组成
* 可以有重叠（是否重叠和滑动距离有关）
* 滑动窗口是固定窗口的更广义的一种形式，滚动窗口可以看做是滑动窗口的一种特殊情况（即窗口大小和滑动距离相等）
* 一个数据可以被（size / slide）个窗口包含

##### 会话窗口（Session Windows）

![image-20210422235825801](https://ww1.sinaimg.cn/large/6af0fe46ly1gpsyqj2a4zj20sq0h4wgk.jpg)

* 由一系列事件组合一个指定时间长度的timeout间隙组成，也就是一段时间没有接收到新数据就会生成新的窗口
* 特点：时间不对齐

### 3.2 Window API

#### 3.2.1 概述

* 窗口分配器 —— `window()` 方法
* **注意：window()方法必须在keyBy之后才能使用**
* 更加简单的方法 `timeWindow()` 定义时间窗口、`countWindow()`定义计数窗口

##### 窗口分配器（Window assigner）

* `window()`方法接收的输入参数是一个WindowAssigner
* WindowAssigner负责将每条输入的数据分发到正确的window中
* Flink提供了通用的WindowAssigner
  * 滚动窗口（tumbling window）
  * 滑动窗口（sliding window）
  * 会话窗口（session window）
  * **全局窗口（global window）**

##### 创建不同类型的窗口

* 滚动时间窗口（tumbling time window）

  `.timeWindow(Time.seconds(15))`

* 滑动时间窗口（sliding time window）

  `.timeWindow(Time.seconds(15),Time.seconds(5))`

* 会话窗口（session window）

  `.window(EventTimeSessionWindows.withGap(Time.minutes(10)))`

* 滚动计数窗口（tumbling count window）

  `.countWindow(5)`

* 滑动计数窗口（sliding count window）

  `.countWindow(10,2)`

PS: DataStream的`windowAll()`类似分区的global操作，是non-parallel的（并行度为1），所有数据都会被传递到同一个算子operator上。官方不推荐使用。

#### 3.2.2 TimeWindow

TimeWindow将指定时间范围内的所有数据组成一个window，一次对一个window里面的素有数据行进计算。

##### 滚动窗口

Flink默认的时间窗口是根据ProcessingTime进行窗口的划分，即将Flink获取到的数据根据进入Flink的时间划分到不通过的窗口中。

##### 滑动窗口

与滚动窗口的函数名一致，只是在传参是需要传入两个参数，windowSize与slidingSize。

#### 3.2.3 CountWindow

 CountWindow根据窗口中相同key元素的数量来触发执行，执行时只计算元素数量达到窗口大小的key对应的结果。

 **PS：CountWindow的window_size指的是相同Key的元素的个数，不是输入的所有元素的总数。**

##### 滚动窗口

 默认的CountWindow是一个滚动窗口，只需要指定窗口大小即可，**当元素数量达到窗口大小时，就会触发窗口的执行**。

##### 滑动窗口

与滚动窗口的函数名一致，只是在传参是需要传入两个参数，windowSize与slidingSize。

**PS:当窗口不足设置的大小时，会先按照步长输出。**
eg: 窗口大小10，步长2，那么前5次输出时，窗口内的元素个数分别是（2，4，6，8，10），再往后就是10个为一个窗口了。

#### 3.2.4 SessionWindow

SessionWindow在给定时间未收到相同key的元素，则新生成一个窗口

`.window(EventTimeSessionWindows.withGap(Time.seconds(5)))`

#### 3.2.5 window function

Window function 定义了要对窗口中收集的数据做的计算操作，主要可以分为两类：

* 增量聚合函数（incremental aggregation functions）
* 全窗口函数（full window functions）

##### 增量聚合函数

* 每条数据到来就进行计算，保持一个简单的状态。（来一条处理一条，但是不输出，到窗口临界位置才输出）
* 典型的增量聚合函数有ReduceFunction，AggregateFunction。

##### 全窗口函数

* 先把窗口所有数据收集起来，等到计算时会遍历所有数据。（来一个存储一个，窗口到临界位置才遍历且计算、输出）
* ProcessWindowFunction，WindowFunction。

#### 其他可选API

* `trigger()` —— 触发器
  定义window什么时候关闭，触发计算并输出结果
* `evitor()` —— 移除器
  定义移除某些数据的逻辑
* `allowedLateness()` ——允许处理迟到的数据
  ![image-20210601173415540](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnfmq5e2vj20h00bngm9.jpg) 
  小圆表示**窗口内的数据**，时间达到窗口endTime时会触发一次窗口计算，迟到的小圆还会分别触发一次窗口计算。
* `sideOutputLateData()` —— 将迟到的数据放入侧输出流
* `getSideOutput()` —— 获取侧输出流
* ![image-20210427192409815](https://ww1.sinaimg.cn/large/6af0fe46ly1gpyj2fhblbj20m20bhdgb.jpg)

## 4. 时间语义和watermark

### 4.1 Flink中的时间语义

![image-20210510233844354](https://ww1.sinaimg.cn/large/6af0fe46ly1gqdrd426xkj20yq0gmabe.jpg)

* **Event Time：事件创建时间**
* Ingestion Time：数据进入Flink时间
* Processing Time：执行操作算子的本地系统时间，与机器相关

*PS: Event Time 是事件创建的时间。它通常由时间中的时间戳描述，例如采集的日志数据中，每一条日志都会记录自己的生成时间，Flink通过时间戳分配器访问事件时间戳。*

### 4.2 EventTime的引入

如果要使用EventTime，那么需要引入EventTime的时间属性，引入方式如下：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 从调用时刻开始给env创建的每一个stream追加时间特征
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

**注：具体的时间，还需要从数据中提取时间戳。**

### 4.3 Watermark

#### 4.3.1 基本概念

数据乱序时，如果只根据eventTime决定window的运行，不能明确数据是否全部到位，但是又不能无限期的等待数据，此时就要有一个机制来保证一个特定时间后，必须触发window进行计算，这个特别的机制就是Watermark。

* Watermark是一种衡量EventTime进展的机制；
* **Watermark是用于处理乱序事件的**，而正确的处理乱序事件，通常用Watermark机制结合Window来实现；
* 数据流中的**Watermark用于表示（timestamp - Watermark）的数据，都已经到达了**，因此，**window的执行也是由Watermark触发的**。
* Watermark可以理解成一个延迟触发机制，可以设置Watermark的延时时长t，每次系统会校验已经达到的数据中最大的maxEventTime，然后认定eventTime小于 `maxEventTime - t` 的所有数据都已经到达，如果有窗口的停止时间为 `maxEventTime - t` ，那么这个窗口就会被触发执行。
  `Watermark = maxEventTime - 延迟时间t`

Flink接收到数据时，会按照一定的规则去生成Watermark，这条Watermark就等于当前所有到达数据中的`maxEventTime - 延迟时长`，也就是说，**Watermark是基于数据携带的时间戳生成的**，一旦Watermark比当前未触发的窗口的停止时间要晚，那么就会触发相应窗口的执行。由于eventTime是由数据携带的，因此，**如果运行过程中无法获取新的数据，那么没有被触发的窗口将永远都不被触发**。

**Flink对迟到的数据有三层保障**，先来后到的保障顺序是：

* Watermark => 约等于放宽窗口标准
* allowedLateness => 允许迟到（窗口已经触发第一次计算，在watermark到达`endOfWindow + allowedLateness`之前每一个到达的数据都会触发一次计算）
* sideOutputLateData => 超过迟到时间，另外捕获，之后可以进行批处理合并先前数据

#### 4.3.2 Watermark的传递

![image-20210530212550480](https://ww1.sinaimg.cn/large/6af0fe46ly1gr0rvi3fwoj20za0l2jsl.jpg)

1. 图一：当前Task中有四个上游Task给自己传输Watermark信息，通过比较Partition WM，只取当前最小值`2`作为本地Task的EventTime clock；上图中，当前Task[0,2)的桶就可以关闭了，因为所有上游中`2`最小，能保证`2`是Watermark是准确的（即所有上游的Watermark都已经>=2）。这时将Watermark=`2`广播到当前Task的下游。
2. 图二：上游的Watermark变动，本地Task的Watermark则变成了`3`，更新本地Task的EventTime clock，同时将最新的Watermark=`3`广播到下游。
3. 图三：上游的Watermark继续更新，但本地Task的最小值仍然为`3`，所以不更新EventTime clock，也不需要广播到下游。
4. 图四：和图二同理，更新本地EventTime clock，同时向下游广播最新的Watermark=`4`。

#### 4.3.3 Watermark的引入

对于**乱序数据**，最常见的引用方式如下：

```java
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

dataStream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
            @Override
            public long extractTimestamp(SensorReading element) {
                return element.getTimestamp() * 1000L;
            }
        });
```

Assigner有两种类型：

* AssignerWithPeriodicWatermarks（常用）
* AssignerWithPunctuatedWatermarks

以上两个接口都继承自TimestampAssigner。

##### Assigner with periodic watermarks

周期性的生成watermark：系统会周期性的将watermark插入到流中（水位线也是一种特殊的事件）。默认周期是200毫秒。可以使用

```java
env.getConfig().setAutoWatermarkInterval(5000);
```

进行设置。

`AssignerWithPeriodicWatermarks`的`getCurrentWatermark()`方法。如果方法返回一个时间戳＞之前水位的时间戳，新的watermark会被插入到流中；反之则不会产生新的watermark。这个检查保证了watermark是单调递增的。

有一种简单的情况，事先得知数据流的时间戳是**单调递增**的，那可以使用`AscendingTimestampExtractor`，这个类会直接使用数据的时间戳生成watermark。

```java
dataStream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<SensorReading>() {
		@Override
		public long extractAscendingTimestamp(SensorReading element) {
      	return element.getTimestamp() * 1000;
		}
});
```

而对于**乱序数据流**，如果能大致估算出数据流中的事件的**最大延迟时间**（🌰为2秒），就使用如下代码：

```java
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

dataStream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
            @Override
            public long extractTimestamp(SensorReading element) {
                return element.getTimestamp() * 1000L;
            }
        });
```

##### Assigner with punctuated watermarks

间断式地生成watermark。和周期性生成的方式不同，这种方式不是固定时间的，而是可以根据需要对每条数据进行筛选和处理。

#### 4.3.4 测试代码

```java
public class WindowTest3EventTimeWindow {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        DataStreamSource<String> dataStreamSource = env.socketTextStream("127.0.0.1", 7777);

        SingleOutputStreamOperator<SensorReading> dataStream = dataStreamSource.map((line) -> {
            String[] splitArr = line.split(",");
            return new SensorReading(splitArr[0], Long.parseLong(splitArr[1]), Double.parseDouble(splitArr[2]));
        })
                // 升序数据设置事件时间和Watermarks
//                .assignTimestampsAndWatermarks(new AscendingTimestampExtractor<SensorReading>() {
//            @Override
//            public long extractAscendingTimestamp(SensorReading element) {
//                return element.getTimestamp() * 1000L;
//            }
//        })
                // 乱序数据设置事件时间和Watermarks
                .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
            @Override
            public long extractTimestamp(SensorReading element) {
                return element.getTimestamp() * 1000L;
            }
        });

        OutputTag<SensorReading> lateOutputTag = new OutputTag<SensorReading>("late") {};

        // 基于事件时间的开窗聚合，统计15秒内温度的最小值
        SingleOutputStreamOperator<SensorReading> result = dataStream.keyBy(SensorReading::getId)
                .timeWindow(Time.seconds(15))
          			// 允许迟到30秒
                .allowedLateness(Time.seconds(30))
                .sideOutputLateData(lateOutputTag)
                .minBy("temperature");

        result.print("minTemperature");
        result.getSideOutput(lateOutputTag).print("late");

        env.execute();
    }
}
```

启动本地socket

```shell
nc -lk 7777
```

输入：

```
// ---------- 1 --------
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718212,34.8
// ---------- 2 --------
sensor_1,1547718200,33.8
sensor_1,1547718203,32.8
sensor_1,1547718204,31.8
sensor_1,1547718205,30.8
// ---------- 3 --------
sensor_1,1547718244,40.8
sensor_1,1547718206,29.8
```

输出：

```
// ---------- 1 --------
minTemperature> SensorReading(id=sensor_1, timestamp=1547718199, temperature=35.8)
minTemperature> SensorReading(id=sensor_7, timestamp=1547718202, temperature=6.7)
minTemperature> SensorReading(id=sensor_6, timestamp=1547718201, temperature=15.4)
minTemperature> SensorReading(id=sensor_10, timestamp=1547718205, temperature=38.1)
// ---------- 2 --------
minTemperature> SensorReading(id=sensor_1, timestamp=1547718200, temperature=33.8)
minTemperature> SensorReading(id=sensor_1, timestamp=1547718203, temperature=32.8)
minTemperature> SensorReading(id=sensor_1, timestamp=1547718204, temperature=31.8)
minTemperature> SensorReading(id=sensor_1, timestamp=1547718205, temperature=30.8)
// ---------- 3 --------
minTemperature> SensorReading(id=sensor_1, timestamp=1547718212, temperature=34.8)
late> SensorReading(id=sensor_1, timestamp=1547718206, temperature=29.8)
```

现象：

1. 第一部分的输出是在第一部分的输入全部输入完成后一次性打印出来的，说明窗口的初始位置为`1547718212 - 2 = 1547718210`，由此可得起始位置为 `1547718195` 为什么不是 `1547718199`？
2. 第二部分 每输入一条 就会打印一条
3. 第三部分 在输入`sensor_1,1547718243,40.7` 就立马打印出一条 
   输入`sensor_1,1547718206,29.8` 以 `late`前缀 打印出一条

分析：

1. 计算窗口的起始位置start和结束位置end
   从`TumblingProcessingTimeWindow`类里的`assignWindows`方法，我们可以得知窗口的起点计算方法如下：`start = timestamp - (timestamp - offset + windowSize) % windowSize` 这里offset为0，所以起始位置`start = 1547718199 - (1547718199 - 0 + 15) % 15 = 1547718195`
2. 30秒内迟到数据更新
   第一部分的数据已经到达`1547718212` 对应的watermark为`1547718210`，已经到达窗口结束位置。第二部分的数据都属于上述窗口内的数据，所以这部分数据为迟到数据，当前任务设置了允许迟到30秒（即Watermark为`1547718240`），所以每来一条窗口内的数据就会进行更新最低温度。
3. 30秒外迟到数据输出侧输出流
   第三部分的第一条数据时间为`1547718244`，watermark为`1547718242`，已经超过了`1547718195`-`1547718210`(`1547718240`)窗口的结束位置及迟到结束位置（即清除位置），也超过了`1547718210`-`1547718225`窗口的结束位置，所以会输出`1547718210`-`1547718225`窗口的结果。第二条数据时间为`1547718206`属于`1547718195`-`1547718210`(`1547718240`)窗口，但现在watermark已经超了该窗口的迟到结束位置，本应该丢弃，但当前任务将其输出到了侧输出流，并以`late`为前缀。

##### 并行任务Watermark传递测试

在上面代码的基础上，修改执行环境并行度为4

启动本地socket

```shell
nc -lk 7777
```

输入：

```
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718211,34
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
```

输出：
上面全部输入完成后，才突然有下面的输出

```
minTemperature:2> SensorReading{id='sensor_10', timestamp=1547718205, temperature=38.1}
minTemperature:3> SensorReading{id='sensor_1', timestamp=1547718199, temperature=35.8}
minTemperature:4> SensorReading{id='sensor_7', timestamp=1547718202, temperature=6.7}
minTemperature:3> SensorReading{id='sensor_6', timestamp=1547718201, temperature=15.4}
```

分析：

1. 为什么上面输入中，最后连续四条相同输入，才触发Window输出结果？
   * Watermark会向子任务广播
     * 在map才设置Wtermark，map根据Rebalance轮询方式分配数据。所以前4个输入分别到4个slot中，4个slot计算得出的Watermark不同（分别为`1547718199-2`，`1547718201-2`，`1547718202-2`，`1547718205-2`）
   * Watermark传递时，会选择当前接收到的最小的一个自己的Watermark
     * 前4次输入中，有些map子任务还没有接收到结束，所有其下游的keyBy后的slot里watermark就是初始值（Long.MIN_VALUE），因为4个上游的Watermark广播最小值就是初始的Long.MIN_VALUE
     * 并行度为4，在最后4个相同的输入，使得Rebalance到4个map子任务的数据`currentMaxTimestamp`都是`1547718212`，通过`getCurrentWatermark()`计算（`currentMaxTimestamp - maxOutOfOrderness`），得到`watermark=1547718210`，所以4个子任务向4个keyBy子任务广播watermark=1547718210，所以keyBy子任务获得的4个上游的Watermark最小值为`1547718210`，更新Watermark为`1547718210`
   * 根据Watermark的定义，我们任务>=Watermark的数据都已经到达。由于此时watermark=>窗口End（`1547718210`），所有window输出结算结果（4个子任务，4个结果）

## 5 Flink状态管理

* 算子状态（Operator State）
* 键控状态（Keyed State）
* 状态后端（State Backends）

### 5.1 状态概述

![image-20210604152412147](https://ww1.sinaimg.cn/large/6af0fe46ly1gsnfn1kpz9j20u80dm445.jpg)

* 由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态
* 可以认为任务状态就是一个本地变量，可以被任务的业务逻辑访问
* **Flink会进行状态管理，包括状态一致性、故障处理以及高效存储和访问，以便于开发人员可以专注于应用程序的逻辑**
* **Flink中，状态始终与特定算子相关联**

总的来说，有两种类型的状态：

* 算子状态（Operator State）
  * 算子状态的作用范围限定为**算子任务**（也就是不能跨任务访问）
* 键控状态（Keyed State）
  * 根据输入数据流中定义的键（Key）来维护和访问

### 5.2 算子状态（Operator State）

![image-20210604160219641](https://ww1.sinaimg.cn/large/6af0fe46ly1gr6amdlx9vj20oc0hmgm7.jpg)

* 算子状态的作用范围限定为算子任务，**同一并行任务**所处理的所有数据都可以访问到相同的状态
* 状态对于**同一任务**而言是共享的（不能跨slot）
* 状态算子不能由相同或不同算子的**另一个任务**访问

#### 算子状态数据结构

* 列表状态（List State）
  将状态表示为一组数据的列表
* 联合列表状态（Union List State）
  也将状态表示数据的列表。它与常规列表状态的区别在于，在发生故障时，或者从保存点（savepoint）启动应用程序时如何恢复
* 广播状态（Broadcast State）
  如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合用广播状态

### 5.3 键控状态（Keyed State）

![image-20210604163217074](https://ww1.sinaimg.cn/large/6af0fe46ly1gr6bhe5mzrj20je0fodgi.jpg)

* 键控状态是根据输入数据流中定义的键（Key）来维护和访问的
* Flink为**每个Key**维护**一个状态实例**，并将具有**相同键的所有数据**，都**分区到同一个算子任务**中，这个任务会维护和处理这个key对应的状态
* **当任务处理一条数据时，Flink会自动将状态的访问范围限定为当前数据的Key**

#### 键控状态数据结构

* 值状态（Value State）
  将状态表示为单个的值
* 列表状态（List State）
  将状态表示为一组数据的列表
* 映射状态（Map State）
  将状态表示为一组key-value对
* 聚合状态（Reducing State & Aggregation State）
  将状态表示为一个用于聚合操作的列表

### 5.4 状态后端（State Backends）

#### 5.4.1 概述

* 每传入一条数据，有状态的算子任务都会读取和更新状态
* 由于有效的状态访问对于处理数据的低延迟至关重要，因此每个并行任务都会在本地维护其状态，以确保快速的状态访问
* 状态的存储、访问以及维护，由一个可插入的组件决定，这个组件就叫做状态后端（State Backend）
* **状态后端主要负责两件事：本地状态管理，以及将检查点（ChekcPoint）状态写入远程存储**

#### 5.4.2 状态后端的类型

* MemoryStateBackend
  * 内存级的状态后端，会将键控状态作为内存中的对象进行管理，将他们存储在TaskManager的**JVM堆**上，而将checkPoint存储在JobManager的**内存**中
  * 特点：快速、低延迟，但不稳定
* FsStateBackend（默认）
  * 将checkPoint存到远程的持久化文件系统（FileSystem）上，而对于**本地状态**，跟 MemoryStateBackend 一样，也会**存在TaskManager的JVM堆**上
  * 同时拥有内存级的本地访问速度，和更好的容错保证
* RocksDBStateBackend
  * 将所有状态序列化，存入本地的RocksDB中存储

## 6 ProcessFunction API（底层API）

之前的**转换算子**是无法访问事件的**时间戳信息和水位线信息**。而在一些应用场景下，极为重要。

DataStream API提供了乙烯类的Low-Level转换算子。可以访问**时间戳、watermark**以及**注册定时事件**。还可以输出**特定的一些事件**，例如超时事件等。<u>Process Funtion用来构建事件驱动的应用以及实现自定义的业务逻辑（使用之前的window函数和装换算子无法实现）。例如，FlinkSQL就是使用Process Funtion实现的。</u>

Flink提供了8个Process Function：

* ProcessFunction
* KeyedProcessFunction
* CoProcessFunction
* ProcessJoinFcuntion
* BroadcastProcessFunction
* KeyedBroadcastProcessFunction
* ProcessWindowFunction
* ProcessAllWindowFunction

![img](https://ww1.sinaimg.cn/large/6af0fe46ly1grawkjonvdj20gp0jtt9g.jpg)

### 6.1 KeyedProcessFunction

相对比较常用的ProcessFunction，根据名字可知是使用在KeyedStream上的。

KeyedProcessFunction用来操作KeyedStream。KeyedProcessFunction会处理流的每一个元素，输出为0个、1个或者多个元素。所有的Process Function都继承自RichFunction接口，所以都有`open()`、`close()`和`getRuntimeContext()`等方法。而`KeyedProcessFunction<K, I, O>`还额外提供了两个方法：

* `processElement(I value, Context ctx, Collector<O> out)`：流中的每一个元素都会调用这个方法，调用结果将会放在Collector数据类型中输出。Context可以访问元素的时间戳，元素的key，以及TimerService时间服务。Context还可以将结果输出到别的流（side outputs）。
* `onTimer(long timestamp, OnTimerContext ctx, Collectior<O> out)`：一个回调函数。当之前注册的定时器触发时调用。`timestamp`为定时器所设定的触发的时间戳。`Collectior`为输出结果的集合。

`OnTimerContext`和`processElement`的Context 参数一样，提供了上下文的一些信息，例如定时器触发的时间信息(事件时间或者处理时间)。

测试代码见：`com.sqlist.apitest.processfunction.ProcessTest1KeyedProcessFunction`

### 6.2 TimerService 和定时器（Timers）

`Context` 和`OnTimerContext` 所持有的TimerService 对象拥有以下方法：

- `long currentProcessingTime()` 返回当前处理时间
- `long currentWatermark()` 返回当前watermark 的时间戳
- `void registerProcessingTimeTimer(long timestamp)` ：会注册当前key的processing time的定时器。当processing time 到达定时时间时，触发timer。
- **`void registerEventTimeTimer(long timestamp)` ：会注册当前key 的event time 定时器。当Watermark水位线大于等于定时器注册的时间时，触发定时器执行回调函数。**
- `void deleteProcessingTimeTimer(long timestamp)` 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。
- `void deleteEventTimeTimer(long timestamp)` 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。

 **当定时器timer 触发时，会执行回调函数onTimer()。注意定时器timer 只能在keyed streams 上面使用。**

### 6.3 侧输出流（SideOutput）

* 大部分DataStream API的算子的输出都是单一输出，也就是某种数据类型的流。只有`split`算子，可以将一条流分成多条流，这些流的数据类型也都相同
* `ProcessFunction`的`side output`功能可以产生多条流，并且这些流的数据类型可以不一样
* 一个`side output`可以定义为`OutputTag[X]`对象，`X`是输出流的数据类型
* `ProcessFunction`可以通过`Context`对象发射一个事件到一个或多个`side output`

### 6.4 CoProcessFunction

* 对于两条输入流，DataStream API提供了`CoProcessFunction`这样的low-level操作。`CoProcessFunction`提供了操作每一个输入流的方法：`processElement1()`和`processElement2()`
* **类似于`ProcessFunctin`，这两种方法都通过`Context`对象来调用。**<u>这个Context对象可以访问事件数据，定时器时间戳，TimerService，以及side outputs。</u>
* **`CoProcessFunction`也提供了`onTimer()`回调函数。**

## 7 容错机制

### 7.1 一致性检查点（CheckPoint）

![image-20210610171905981](https://ww1.sinaimg.cn/large/6af0fe46ly1grdal7n67fj20e209iaa7.jpg)

* Flink故障恢复机制的核心，就是**应用状态的一致性检查点**
* 有状态流应用的一致性检查点，其实就是**所有任务的状态**，在某个时间点的一份拷贝（一份快照）；**这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时候**
  （上图中 `5` 这个数虽然进了奇数流，但是偶数流也应该做快照，因为属于同一个相同数据，只是没有被偶数流处理）
  （这里根据奇偶性分流，偶数流求偶数和，奇数流求奇数和，`5`明显已经被`sum_odd(1+3+5)`处理了，且`sum_even`不需要处理该数据，因为前面已经判断该数据不需要到`sum_even`流，相当于所有任务都已经处理完`source`的数据`5`）
* 在`JobManager`中也有个CheckPoint的指针，指向了仓库的状态快照的一个拓补图，为以后的数据故障恢复做准备

### 7.2 从检查点恢复状态

#### 7.2.1 故障

* 在流应用执行期间，Flink会定期保存状态的一致性检查点
* 如果发生故障，Flink将会使用最近的检查点来恢复应用程序的状态，并重新启动处理流程
  *PS:新版Flink，可以配置重启不重启全部task，只重启发生故障的部分*

![image-20210610173515842](https://ww1.sinaimg.cn/large/6af0fe46ly1grdb0t6mfyj20g904zwee.jpg)

如图所示，`7`被`source`读到后，在传给`sum_odd`时，`sum_odd`宕机了，数据传输发生中断

#### 7.2.2 恢复-第一步

* 重启应用

![image-20210610174425767](https://ww1.sinaimg.cn/large/001XqbP0ly1grdbabs89aj60g9057glj02.jpg)

如图所示，重启应用后，起初流都是空的

#### 7.2.3 恢复-第二步

* 从checkPoint中读取状态，将状态重置
* 从checkPoint重启应用程序后，其内部状态与检查点完成时的状态完全相同

![image-20210610174800021](https://ww1.sinaimg.cn/large/6af0fe46ly1grdbe1gtypj20fs09at8w.jpg)

读取在远程仓库（Storage，这里的仓库指状态后端保存数据指定的三种方式之一）保存的状态

#### 7.2.4 恢复-第三步

* 消费并处理 从检查点开始到发生故障之间的所有数据
* **这种检查点的保存和恢复机制可以为应用程序提供“精确一次”（exactly-once）的一致性，因为所有算子都会保存检查点并恢复其状态，这样一来所有的输入流都会被重置到检查点完成时的位置**
  *PS:这里要求`source`源也能记录状态，回退到读取数据`7`的状态，kafka有相应的偏移指针能完成*

### 7.3 Flink检查点算法

#### 7.3.1 概述

`Checkpoint`和`Watermark`一样，都会以广播的形式告诉所有下游

---------

* 基于Chandy-Lamport算法的分布式快照
* **将检查点的保存和数据处理分离开，不暂停整个应用**
  （就是每个人物单独记录自己的快照到内存，之后再到`JobManager`整合）

----

* 检查点分界线（Checkpoint Barrier）
  * Flink的检查点算法用到一种称为分界线（barrier）的特殊数据形式，用来把一条流上数据按照不同的检查点分开
  * **分界线之前到来的数据导致的状态更改，都会被包含在当前分界线所属的检查点中；而基于分界线之后的数据导致的所有更改，都会被包含在之后的检查点中**

#### 7.3.2 讲解

![image-20210610192918698](https://ww1.sinaimg.cn/large/6af0fe46ly1grdebg5qflj20i308gglt.jpg)

* 现在有一个两个输入流的应用程序，用并行的两个`Source`任务来读取
* 两条流是自然数数据流，蓝色数据流已经输出`蓝3`，黄色数据流已经输出`黄4`
* `Source`端`Source1`接收到了数据`蓝3`，正在往下游发送数据`蓝2和蓝3`；`Source2`接收到了数据`黄4`，正在往下游发送数据`黄4`
* `Sum even`已经处理完`黄2`，状态显示为`2`，并正在向下游发送`2`；`Sum odd`已经处理完`黄1、蓝1、黄3`，状态显示为`5`，并正在向下游发送`5和2`

-----

![image-20210610194143201](https://ww1.sinaimg.cn/large/6af0fe46ly1grdeod39flj20il08caa8.jpg)

* **`JobManager`启动检查点，向每个`Source`发送一条带新检查点ID的消息**

这个带有新检查点ID的信息称为`barrier`，图中由三角形表示，数值2只是ID

-----

![image-20210610194745132](https://ww1.sinaimg.cn/large/6af0fe46ly1grdeunwmlgj20gl09hjrn.jpg)

* `Source`将他们的状态写入`remote storage`，并向每个下游发出检查点barrier
* `State Backend`在状态存入`remote storage`后，会返回通知`Source`任务，`Source`会向`JobManager`确认检查点完成

如图，`Source`端接收到`barrier`后，将自己的状态`3和4`写入`remote storage`，并且向`JobManager`发送`checkPoint`成功，然后向每个下游发送一个检查点`barrier`
（*可以看出在Source接受barrier时，数据流也在不断的处理，不会进行中断*）

此时`Sum even`已经处理了`蓝2`变成了`4`，向下游发送了`4`；`Sum odd`已经处理完`蓝3`变成了`8（黄1+蓝1+黄3+蓝3）`，并向下游发送`8`

此时，检查点`barrier`都还未到`Sum odd`和`Sum even`

----

![image-20210610200441867](https://ww1.sinaimg.cn/large/6af0fe46ly1grdfc9uomkj20fl0973ys.jpg)

* **分界线对齐：barrier向下游传递，`sum`会等待所有上游分区的barrier到达**
* **对于barrier已经达到的分区，后续到达的数据会被缓存**
* **而barrier尚未到达的分区，数据会被正常处理**

图中，蓝色流的barrier先到达了`sum even`，黄色流的barrier还未到；因为`Source`数据不中断，一直在进行处理，`蓝4`先与黄色流的barrier到达`Sum even`，此时，`蓝4`会被缓存，继续等待黄色流barrier到来

---

![image-20210610222519259](https://ww1.sinaimg.cn/large/6af0fe46ly1grdjem87xwj20h808faag.jpg)

* 当收到所有上游分区的barrier时，任务就将其状态保存到`State Backend`中，然后向下游发送barrier

*当蓝色的barrier和黄色的barrier(所有分区的)都到达后，进行状态保存到远程仓库，**然后对JobManager发送消息，说自己的检查点保存完毕了***

---

![image-20210610222754699](https://ww1.sinaimg.cn/large/6af0fe46ly1grdjha1jrxj20ho081aa8.jpg)

* 向下游发送 barrier 后，任务继续正常的数据处理

`Sum even`将原本缓存的`蓝4`进行处理，又将到达的`黄6`进行处理

---

![image-20210610222937317](https://ww1.sinaimg.cn/large/6af0fe46ly1grdjj1rwumj20h308ft8x.jpg)

* `Sink`向`JobManager`确认状态保存到`State Backend`
* 当所有任务都确认已成功将状态保存到`State Backend`时，检查点就真正完成了

### 7.4 保存点（SavePoint）

**CheckPoint为自动触发保存，SavePoint为手动触发保存**

* Flink还提供了可以自定义的镜像保存功能，就是保存点（SavePoint）
* 原则上，创建保存点使用的算法与检查点完全相同，因此保存点可以认为就是具有一些额外源数据的检查点
* Flink不会自动创建保存点，因此用户（或者外部调度程序）必须明确地触发创建操作
* 保存点是一个强大的功能。除了**故障恢复**外，保存点可以用于：**有计划的手动备份、更新应用程序、版本迁移、暂停和重启程序**等

### 7.5 检查点和重启策略配置

```java
// 检查点配置
env.enableCheckpointing(300);

// 高级选项
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// CheckoutPoint的处理超时时间
env.getCheckpointConfig().setCheckpointTimeout(60000L);
// 最大允许同时处理几个CheckPoint
env.getCheckpointConfig().setMaxConcurrentCheckpoints(2);
// 这个时间间隔是 当前checkpoint的处理完成时间与接收最新一个checkpoint之间的时间间隔
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(100L);
// 如果同时开启了savepoint且有更新的备份，是否倾向于使用更老的自动备份checkpoint来恢复，默认false
env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
// 最多能容忍几次checkpoint处理失败（默认0，即checkpoint处理失败，就当作程序执行异常）
env.getCheckpointConfig().setTolerableCheckpointFailureNumber(0);

// 3. 重启策略配置
// 固定延迟重启（最多尝试3次，每次间隔10s）
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 10000L));
// 失败率重启（在10分钟内最多尝试3次，每次至少间隔1分钟）
env.setRestartStrategy(RestartStrategies.failureRateRestart(3, Time.minutes(10), Time.minutes(1)));
```

### 7.6 状态一致性

#### 7.6.1 概述

![image-20210613184404765](https://ww1.sinaimg.cn/large/6af0fe46ly1grgxy184iaj20z609ogm3.jpg)

* 有状态的流处理，内部每个算子任务都可以有自己的状态
* 对于流处理器内部来说，所谓的状态一致性，其实就是我们所说的计算结果要保证准确
* 一条数据不应该丢失，也不应该重复计算
* 在遇到故障时可以恢复状态，恢复之后重新计算，结果也应该是完全正确的

#### 7.6.2 分类

**Flink的一个重大价值在于，它既保证了`exactly-once`，也具有低延迟和高吞吐的处理能力**

* `AT-MOST-ONCE（最多一次）`：当任务故障时，最简单的做法是什么都不干，既不恢复丢失的状态，也不重播丢失的数据。At-most-once 语义的含义是最多处理一次事件。

  *这其实是没有正确性保障的委婉说法——故障发生之后，计算结果可能丢失。类似的比如网络协议的udp。*

* `AT-LEAST-ONCE（至少一次）`：在大多数的真实应用场景，我们希望不丢失事件。这种类型的保障称为 at-least-once，意思是所有的事件都得到了处理，而一些事件还可能被处理多次。

  *这表示计数结果可能大于正确值，但绝不会小于正确值。也就是说，计数程序在发生故障后可能多算，但是绝不会少算。*

* `EXACTLY-ONCE（精确一次）`：**恰好处理一次是最严格的保证，也是最难实现的。恰好处理一次语义不仅仅意味着没有事件丢失，还意味着针对每一个数据，内部状态仅仅更新一次。**

  *这指的是系统保证在发生故障后得到的计数结果与正确值一致。*

#### 7.6.3 一致性检查点（CheckPoints）

* Flink使用一种轻量级快照机制——检查点（CheckPoint）来保证`exactly-once`语义
* 有状态流应用的一致性检查点，其实就是：所有任务的状态，在某个时间点的一份备份（一份快照），而这个时间点，**应该是所有任务都恰好处理完一个相同的输入数据的时间**
* 应用状态的一致性检查点，是**Flink故障恢复机制的核心**

##### 端到端（end-to-end）状态一致性

* 前面所讲的一致性保证都是由流处理器实现的，也就是在**Flink流处理器内部保证**；而在真实场景中，流处理应用除了**流处理器**以外包含了**数据源（Kakfa）**和输出到**持久化系统**
* 端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终；每一个组件都保证了它自己的一致性
* **整个端到端的一致性级别取决于所有组件中一致性最弱的组件**

##### 端到端 exactly-once

* 内部保证——checkpoint
* source端——可重设数据的读取位置
* sink端——从故障恢复时，数据不会重复写入外部系统
  * 幂等写入
  * 事务写入

###### 幂等写入

* 一个操作，可以重复执行很多次，但只导致一次结果更改，也就是说，后面再重复执行就不起作用了
  *（中间可能会存在不正确的情况，只能保证最后结果正确（最终一致性）。比如5=>10=>15=>5=>10=>15，虽然最后是恢复到了15，但是中间有个恢复的过程，如果这个过程能够被读取，就会出问题。）*

![image-20210613213303226](https://ww1.sinaimg.cn/large/6af0fe46ly1grgyr4cr6uj20pc09s3yi.jpg)

###### 事务写入

* 事务
  * 所有操作必须成功完成，否则在每个操作中所作的所有更改都会被撤销
  * 具有原子性：一个事务中的一系列的操作要么全部成功，要么一个都不成功
* Flink中实现思想
  * **构建的事务对应着checkpoint，等到checkpoint真正完成的时候，才把所有对应的结果写入sink系统中**。
* 实现方式
  * 预写日志（WAL）
  * 两阶段提交（2PC）

###### 预写日志（Write-Ahead-Log，WAL）

* 把结果数据先当成状态保存，然后在收到checkpoint完成的通知时，一次性写入sink系统
* 简单易于实现，由于数据提前在状态后端中做了缓存，所以无论什么sink系统，都能用这种方式一批搞定
* 缺点：批处理，需要等待，增大系统延迟；写入时，如果只写入一半，导致失败，就需要重放，重放时就可能会导致一个属于写入多次（外部系统写入的时候不能保证原子性）
* DataStream API提供了一个模版类：GenericWriteAheadSink，来实现这种事务性sink

###### 两阶段提交（Two-Phase-Commit，2PC）

* 对于每个checkpoint，sink任务会启动一个事务，并将接下来所有接收到的数据添加到这个事务中，将这些数据写入外部sink系统，但不提交——这时只是“预提交”
* 只有在checkpoint真正完成时（sink收到`Jobmanager`返回checkpoint成功时），才正式提交，实现结果的真正写入
* **这种方式真正实现了exactly-once，它需要一个提供事务支持的外部sink系统**。Flink提供了TwoPhaseCommitSinkFunction接口

###### 2PC对外部sink系统的要求

* 外部sink系统必须提供事务支持，或者sink任务必须能够模拟外部系统上的事务
* 在`CheckPoint`的间隔期间里，必须能够开启一个事务并接受数据的写入
* 在收到`CheckPoint`完成的通知之前，事务必须是`等待提交`的状态（数据还不能消费，Kafka有隔离级别的概念）。在故障恢复的情况下，这可能需要一点时间。如果这个时候`sink`系统关闭事务（或者超时等），那么未提交的数据就会丢失
* `sink`必须能够在进程失败后恢复事务
* 提交事务必须是幂等操作

###### 不同Source和Sink的一致性保证

|    sink\source    |   不可重置   |                          可重置                           |
| :---------------: | :----------: | :-------------------------------------------------------: |
|    任意（Any）    | At-most-once |                       At-least-once                       |
|       幂等        | At-most-once |     Exactly-once<br />（故障恢复时会出现短暂不一致）      |
|  预写日志（WAL）  | At-most-once | At-least-once<br />（绝大多数情况下能够保证Exactly-once） |
| 两阶段提交（2PC） | At-most-once |                       Exactly-once                        |

### 7.7 Flink+Kafka端到端状态一致性的保证

* 内部——利用`CheckPoint`机制，把状态存盘，发生故障的时候可以恢复，保证内部的状态一致性
* source——`kafka consumer`作为`source`，可以将偏移量保存下来，如果后续任务出现了故障，恢复时候可以由连接器重制偏移量，重新消费数据，保证一致性
* `sink`——`kakfa producer`作为`sink`，采用两阶段提交`sink`，需要实现一个`TwoPhaseCommitSinkFunction`

#### 7.7.1 Exactly-once 两阶段提交

![image-20210614141202007](https://ww1.sinaimg.cn/large/6af0fe46ly1grhrmkhtjuj20yo0g4aan.jpg)

* `JobManager`协调各个`TaskManager`进行`CheckPoint`存储
* `CheckPoint`保存在`StateBackend`中，默认`StateBackend`是内存级，也可以改成文件集的进行持久化保存

![image-20210614141802776](https://ww1.sinaimg.cn/large/6af0fe46ly1grhrst71csj20y20fgt9f.jpg)

* 当`CheckPoint`启动时，`JobManager`会将检查点分界线（barrier）注入数据流
* barrier会在算子间传递下去

![image-20210614141927573](https://ww1.sinaimg.cn/large/6af0fe46ly1grhrvudf1bj20y20h8my2.jpg)

* 每个算子会对当前的状态做个快照，保存到`StateBackend`
* `CheckPoint`机制可以保证Flink内部状态的一致性

![image-20210614215633064](https://ww1.sinaimg.cn/large/6af0fe46ly1gri5c7xsiuj20y80he0tz.jpg)

* 每个内部的`transform`任务遇到`barrier`时，都会吧状态存到`StateBackend`
* `sink`任务首先把数据写入到外部kafka，**这些数据都属于预提交的事务；遇到`barrier`时，把状态保存到`StateBackend`，并开启新的预提交事务
  *（数据流中，`barrier`之前的数据还是处在之前的事务中，遇到`barrier`之后的数据另外开启一个新的事务）*

![image-20210614220544404](https://ww1.sinaimg.cn/large/001XqbP0ly1gri5ccifzrj60y20esjsa02.jpg)

* 当所有算子任务的`CheckPoint`完成时，`JobManager`会向所有任务发送通知，确认这次`CheckPoint`完成
* `Sink`任务收到确认通知后，正式提交之前的事务，kafka中未确认的数据改为“已确认”

#### 7.7.2 步骤总结

1. 第一条数据来了之后，`sink`任务开启一个 kafka 的事务（transaction），正常写入 kafka 分区日志但标记为未提交，这就是“预提交”
2. `JobManager` 触发 `CheckPoint` 操作，`barrier` 从 `source` 开始向下传递，遇到 `barrier` 的算子将状态存入状态后端，并通知 `JobManager`
3. `sink` 收到 `barrier`，保存当前状态，存入 `barrier`，通知 `JobManagers`，并开启下一阶段的事务，用于提交下个检查点的数据
4. `JobManager` 收到所有任务的通知，发出确认信息，表示 `CheckPoint` 完成
5. `Sink` 任务收到 `JobManager` 的确认信息，正式提交这段时间的数据
6. 外部kafka关闭事务，提交的数据可以正常消费

#### 7.7.3 配置要求

* Kakfa需要打开事务配置
* kafka事务的超时时间需要与Flink的`CheckPoint`的超时时间相匹配
*  kafka的消费者隔离级别需要设置为`read commited`（默认为`read uncommited`）

## 8 Table API 与 SQL

### 8.1 概述

* Flink对批处理和流处理，提供了统一的上层API
* Table API 是一套内嵌在Java和Scala的查询API，它允许以非常直观的方式组合来自一些关系运算符的查询
* Flink的SQL支持基于实现了SQL标准的`Apache Calcite`

![image-20210628100536883](https://ww1.sinaimg.cn/large/6af0fe46ly1grxr6hekydj20h205vq32.jpg)

* Pom依赖

  ```xml
  <!-- Table API 和 Flink SQL -->
  <dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner-blink_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
  </dependency>
  ```

###  8.2 基本程序结构

```Java
StreamTableEnvironment tableEnv = ... // 创建表的执行环境

// 创建一张表，用于读取数据
tbaleEnv.connect(...).createTemporaryTable("inputTable");

// 注册一张表，用于将结果输出
tbaleEnv.connect(...).createTemporaryTable("outputTable");

// 通过Table API 查询算子，得到一张结果表
Table resultTable = tableEnv.from("inputTable").select(...);

// 通过SQL查询语句，得到一张结果表
Table sqlResult = tableEnv.sqlQuery("SELECT ... FROM inputTable ...");

// 将结果表写入输出表中
// result.insertInto("outputTable");
result.executeInsert("outputTable");
```

### 8.3 Table API批处理和流处理

新版本Blink，真正把批处理、流处理都以DataStream实现

* 创建环境-样例代码

```java
/**
 * 1.1 基于 老版本 planner 的 流处理
 * Flink Stream
 */
StreamExecutionEnvironment fsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
EnvironmentSettings fsEnvSettings = EnvironmentSettings.newInstance()
  .inStreamingMode()
  .useOldPlanner()
  .build();
StreamTableEnvironment fsTableEnv = StreamTableEnvironment.create(fsEnv, fsEnvSettings);
// TableEnvironment fsTableEnv = TableEnvironment.create(fsEnvSettings);

/**
 * 1.2 基于 老版本 planner 的 批处理
 * Flink Batch
 */
ExecutionEnvironment fbhEnv = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment fbTableEnv = BatchTableEnvironment.create(fbhEnv);

/**
 * 1.3 基于 blink 的 流处理
 * Blink Stream
 */
StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
EnvironmentSettings bsEnvSettings = EnvironmentSettings.newInstance()
  .useBlinkPlanner()
  .inStreamingMode()
  .build();
StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsEnvSettings);
// TableEnvironment bsTableEnv = TableEnvironment.create(bsEnvSettings);

/**
 * 1.4 基于 blink 的 批处理
 * Blink Batch
 */
EnvironmentSettings bbEnvSettings = EnvironmentSettings.newInstance()
  .useBlinkPlanner()
  .inBatchMode()
  .build();
TableEnvironment bbTableEnv = TableEnvironment.create(bbEnvSettings);
```

#### 8.3.1 表（Table）

* `TableEnvironment`可以注册目录`Catalog`，并可以基于`Catalog`注册表
* **表（Table）是由一个“标识符”（identifier）来指定的，由3部分组成：Catalog名、数据库（database）和对象名**
* 表可以是常规的，也可以是虚拟的（视图，View）
* 常规表（Table）一般可以用来描述外部数据，比如文件、数据库表或消息队列的数据，也可以直接从`DataStream`转换而来
* 视图（View）可以从现有的表中产生，通常是table API 或者SQL查询的一个结果集

#### 8.3.2 创建表

* `TableEnvironment`可以调用`connect()`方法，连接外部系统，并调用`.createTemporaryTable()`方法，在`Catalog`中注册表

  ```java
  tableEnv.connect(...)
    .withFormat(...)
    .withSchema(...)
    .createTemporaryTable("table");
  ```

#### 8.3.3 创建TableEnvironment

>[FLINK 1.12.2 中的TableEnvironment](https://blog.csdn.net/arwenlin/article/details/116741539)

* 创建表的执行环境，需要将flink流处理的执行环境传入

```java
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
```

* TableEnvironment是flink中集成Table API和SQL的核心概念，所有对表的操作都基于TableEnvironment
  - 注册Catalog
  - 在Catalog中注册表
  - 执行SQL查询
  - 注册用户自定义函数（UDF）

##### 测试代码

```java
com.sqlist.apitest.tablesql.TableSqlTest2CommonApi
```

#### 8.3.4 表的查询

* Table API 是集成在Scala和java的查询API

* Table API基于代表"表"的Table类，并提供一整套操作处理的方法API；这些方法会返回一个新的Table对象，表示对输入表应用转换操作的结果

* 有些关系型转换操作，可以由多个方法调用组成，构成链式调用结构

  ```java
  Table sensorTable = tableEnv.from("sensor");
  Table resultTable = sensorTable
    .select("id","temperature")
    .filter("id = 'sensor_1'");
  ```

##### 从文件获取数据

```java
com.sqlist.apitest.tablesql.TableSqlTest2CommonApi
```

* 部分输出结果
  里面的`false`表示上一条保存的记录被删除，true则是新加入的数据
  所以Flink的Table API在更新数据时，实际是先删除原本的数据，再添加新数据。

```
sqlAggTable> (true,sensor_6,1,15.4)
sqlAggTable> (true,sensor_7,1,6.7)
sqlAggTable> (true,sensor_10,1,38.1)
sqlAggTable> (true,sensor_1,1,35.8)
sqlAggTable> (false,sensor_1,1,35.8)
sqlAggTable> (true,sensor_1,2,42.9)
sqlAggTable> (false,sensor_1,2,42.9)
sqlAggTable> (true,sensor_1,3,38.6)
sqlAggTable> (false,sensor_1,3,38.6)
sqlAggTable> (true,sensor_1,4,46.45)
sqlAggTable> (false,sensor_1,4,46.45)
sqlAggTable> (true,sensor_1,5,41.160000000000004)
```

##### 数据写入到文件

写入到文件有局限性，只能是批处理，且只能是追加写，不能是更新式的随机写。

```
com.sqlist.apitest.tablesql.TableSqlTest2CommonApi
```

#### 8.3.5 更新模式

* 对于流式查询，需要声明如何在表和外部连接器之间执行转换
* 与外部系统交换的消息类型，由更新模式（Update Mode）指定
* 追加（Append）模式
  * 表只做插入操作，和外部连接器只交换插入（Insert）消息
* 撤回（Retract）模式
  * 表和外部连接器交换添加（Add）和撤回（Retract）消息
  * 插入操作（Insert）编码为Add消息；删除（Delete）编码为Retract消息；**更新（Update）编码为上一条的Retract和下一条的Add消息**
* 更新插入（Upsert）模式
  * 更新和插入都被编码为Upsert消息；删除编码为Delete消息

### 8.4 表和流的转换（有界的）

#### 8.4.1 将Table转换成DataStream

* 表可以转换为DataStream或DataSet，这样自定义流处理或批处理程序就可以继续在 Table API 或 SQL 查询的结果上运行了

* 将表转换为 DataStream 或 DataSet 时，需要指定生成的数据类型，即要将表的每一行转换成的数据类型

* 表作为流式查询的结果，是动态更新的

* 转换有两种转换模式：追加（Appende）模式和撤回（Retract）模式

* 追加模式

  * 用于表只会被插入（Insert）操作更改的场景

    ```java
    DataStream<Row> resultStream = tableEnv.toAppendStream(resultTable,Row.class);
    ```

* 撤回模式

  * 用于任何场景，有些类似于更新模式中的Retract模式，它只有Insert和Delete操作
  * 得到的数据会增加一个Boolean类型的标志位（返回的第一个字段），用它来表示到底是新增的数据（Insert），还是被删除的数据（Delete）。
    *（更新数据，先删除旧数据，再插入新数据）*

#### 8.4.2 将DataStream转换成Table

* 对于一个DataStream可以直接转换成Table，进而方便地调用Table API做转换操作

  ```java
  DataStream<SensorReading> dataStream = ...
  Table sensorTable = tableEnv.fromDataStream(dataStream);
  ```

* 默认转换后的Table schema和DataStream中的字段定义一一对应，也可以单独指定出来

  ```java
  DataStream<SensorReading> dataStream = ...
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, timestamp as ts, temperature");
  ```

#### 8.4.3 创建临时视图（Temporary View）

* 基于DataStream创建临时视图

  ```java
  tableEnv.createTemporaryView("sensorView",dataStream);
  tableEnv.createTemporaryView("sensorView",
                              dataStream, "id, timestamp as ts, temperature");
  ```

* 基于Table创建临时视图

  ```java
  tableEnv.createTemporaryView("sensorView", sensorTable);
  ```

### 8.5 查看执行计划

* Table API 提供了一种机制来解释计算表的逻辑和优化查询计划

* 查看执行计划，可以通过`TableEnvironment.explain(table)`方法或`TableEnvironment.explain()`方法完成，返回一个字符串，描述三个计划

  - 优化的逻辑查询计划
  - 优化后的逻辑查询计划
  - 实际执行计划

  ```java
  String explaination = tableEnv.explain(resultTable);
  System.out.println(explaination);
  ```

###  8.6 流处理和关系代数的区别（无界的）

Table API 和 SQL，本质上还是基于关系表的操作方式；而关系型表、关系代数，以及SQL本身，一般是有界的，更适合批处理的场景。这就导致在进行流处理的过程中，理解会稍微复杂一些，需要引入一些特殊概念。

|                           |    关系代数（表）/ SQL     |                    流处理                    |
| :-----------------------: | :------------------------: | :------------------------------------------: |
|      处理的数据对象       |     字段元组的有界集合     |              字段元组的无限序列              |
| 查询（Query）对数据的访问 |  可以访问到完整的数据输入  |   无法访问所有数据，必须持续“等待流式输入”   |
|       查询终止条件        | 生成固定大小的结果集后终止 | 永不停止，根据持续收到的数据不断更新查询结果 |

### 8.6.1 动态表（Dynamic Tables）

我们可以**随着新数据的到来，不停地在之前的基础上更新结果**。这样得到的表，在Flink Table API概念里，就叫做“动态表”（Dynamic Tables）。

* 动态表是Flink对流数据的Table API 和 SQL 支持的核心概念
* 与表示批处理数据的静态表不同，动态表是随时间变化的
* 持续查询（Continuous Query）
  * 动态表可以像静态的批处理表一样进行查询，查询一个动态表会产生**持续查询（Continuous Query）**
  * **持续查询永远不会终止，并会生成另一个动态表**
  * 查询（Query）会不断更新其动态结果表，以反映其动态输入表上的更改。

####  8.6.2 动态表和持续查询

![image-20210706192842783](https://ww1.sinaimg.cn/large/6af0fe46ly1gsebof3okxj20np04774o.jpg)

流式表查询的处理过程：

1. 流被转换成动态表
2. 对动态表计算**持续查询**，生成新的动态表
3. 生成的动态表被转换回流

####  8.6.3 将流转换成动态表

* 为了处理带有关系查询的流，必须先将其转换为表
* 从概念上讲，流的每个数据记录，都被解释为对结果表的插入（Insert）操作

*本质上，我们其实是从一个、只有插入操作的changlog（更新日志）流，来构建一个表*

#### 8.6.4 持续查询

* 持续查询，会在动态表上做计算处理，并作为结果生成新的动态表

  *与批处理查询不同，连续查询从不终止，并根据输入表上的更新更新其结果表。*

  *在任何时间点，连续查询的结果在语义上，等同于 在输入表的快照上，以批处理模式执行的同一查询的结果。*

![image-20210706194556456](https://ww1.sinaimg.cn/large/6af0fe46ly1gsebocl3bfj20j50a2aa8.jpg)

#### 8.6.5 将动态表转换成DataStream

* 与常规的数据库表一样，动态表可以通过插入（Insert）、更新（Update）和删除（Delete）更改，进行持续的修改
* **将动态表转换为流或将其写入外部系统时，需要对这些更改进行编码**

----

* 仅追加流（Append-only）

  * 仅通过插入（Insert）更改来修改的动态表，可以直接转换为仅追加流

* 撤回流（Retreact）

  * 撤回流式包含两类消息的流：添加（Add）消息和撤回（Retract）消息

  *动态表通过将Insert消息编码为add消息，Delete编码为Retract消息，Update编码为被更改行（前一行）的retract消息和更新后行（新行）的add消息，转换为retract流*

* Upsert（更新插入流）

  * Upsert流也包含两种类型的消息：Upsert消息和Delete消息

  *通过将Insert和Update更改编码为Upsert消息，将Delete更改编码为Delete消息，就可以将具有唯一键（Unique key）的动态表转换为流*

![image-20210706200920062](https://ww1.sinaimg.cn/large/6af0fe46ly1gsebo8inflj20ma0dh74n.jpg)

## 9 时间特性（Time Attributes）

### 9.1 概述

* 基于时间的操作（比如Table API 和 SQL 中的窗口操作），需要定义相关的时间语义和时间数据来源的信息
* Table可以提供一个逻辑上的时间字段，用于在表处理程序中，指示时间和访问相应的时间戳
* **时间属性，可以是每个表schema的一部分。一旦定义了时间属性，它就可以作为一个字段引用，并且可以在基于时间的操作中使用**
* 时间属性的行为类似于常规时间戳，可以访问，并且进行计算

### 9.2 定义处理时间（Processing Time）

* PorcessingTime语义下，允许表处理程序根据机器的本地时间生成结果。它是时间的最简单概念。它既不需要提取时间戳，也不需要生成watermark

#### 9.2.1 由DataStream转换成表时指定

* 在定义Table Schema期间，可以使用`.proctime`，指定字段名定义处理时间字段

* **这个`proctime`属性只能通过附加逻辑字段，来扩展物理schema。因此，只能在schema定义的末尾定义它**

  ```java
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, temperature, pt.proctime");
  ```

#### 9.2.2 定义Table Schema时指定

```java
// 暂时不能使用！ 会报错！
tableEnv.connect(new FileSystem().path(filePath))
                .withFormat(new Csv())
                .withSchema(new Schema()
                        .field("id", DataTypes.STRING())
                        .field("timestamp", DataTypes.BIGINT())
                        .field("temp", DataTypes.DOUBLE())
                        .field("pt", DataTypes.TIMESTAMP(3)).proctime())
                .createTemporaryTable("sensor");
```

#### 9.2.3 创建表的DDL中定义

```java
// ddl 方式  只有blink环境可以用
        String createDdl =
                "create table sensor (" +
                " id varchar(20) not null," +
                " ts bigint," +
                " temperature double," +
                " pt AS PROCTIME()" +
                ") with (" +
                " 'connector.type' = 'filesystem'," +
                " 'connector.path' = '/Users/sqlist/Project/java/flink/demo/src/main/resources/sensor'," +
                " 'format.type' = 'csv'" +
                ")";
        tableEnv.executeSql(createDdl);
```

#### 9.2.4 测试代码

```java
com.sqlist.apitest.tablesql.TableSqlTest3ProcTime
```

### 9.3 定义事件时间（Event Time）

* Event Time 语义，允许表处理程序根据每个记录中包含的时间生成结果。这样即使在有乱序事件或者延迟事件时，也可以获得正确的结果
* **为了处理无序事件，并区分流中的准时和迟到事件，Flink需要从事件数据中，提取时间戳，并用来推送事件时间的进展**

#### 9.3.1 由DataStream转换成表时指定

- 由DataStream转换成表时指定（推荐）
- 在DataStream转换成Table，使用`.rowtime`可以定义事件事件属性

```java
SingleOutputStreamOperator<SensorReading> sensorDataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        }).assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
            @Override
            public long extractTimestamp(SensorReading element) {
                return element.getTimestamp() * 1000L;
            }
        });

// 将DataStream转换为Table，并指定时间字段
Table table1 = tableEnv.fromDataStream(sensorDataStream, $("id"), $("timestamp").rowtime(), $("temperature"));
// 或者，直接追加时间字段
Table table2 = tableEnv.fromDataStream(sensorDataStream, $("id"), $("timestamp"), $("temperature"), $("rt").rowtime());
```

#### 9.3.2 定义Table Schema时指定

```java
// 定义Table Schema时指定
tableEnv.connect(new FileSystem().path(filePath))
  .withFormat(new Csv())
  .withSchema(new Schema()
              .field("id", DataTypes.STRING())
              .field("timestamp", DataTypes.BIGINT())
              .rowtime(
                new Rowtime()
                .timestampsFromField("timestamp")
                .watermarksPeriodicBounded(1000))
              .field("temp", DataTypes.DOUBLE()))
  .createTemporaryTable("sensor");

Table sensorTable1 = tableEnv.from("sensor");
```

#### 9.3.3 创建表的DDL中定义

```java
// 创建表的DDL中定义 blink支持 
String createDdl =
  "create table sensor2 (" +
  " id varchar(20) not null," +
  " ts bigint," +
  " temperature double," +
  " rt AS TO_TIMESTAMP( FROM_UNIXTIME(ts) )," +
  " watermark for rt as rt - interval '1' second" +
  ") with (" +
  " 'connector.type' = 'filesystem'," +
  " 'connector.path' = '/Users/sqlist/Project/java/flink/demo/src/main/resources/sensor'," +
  " 'format.type' = 'csv'" +
  ")";
tableEnv.executeSql(createDdl);
```

#### 9.3.4 测试代码

```java
com.sqlist.apitest.tablesql.TableSqlTest4EventTime
```

### 9.4 窗口

* 时间语义，要配合窗口操作才能发挥作用
* 在Table API 和SQL 中，主要有两种窗口
  * `Group Windows`分组窗口
    * **根据时间戳或行计数 间隔，将行聚合到有限的组中，并对每个组的数据执行一次聚合函数**
  * `Over Windows`
    * 针对每个输入行，计算相邻行范围内的聚合

#### 9.4.1 Group Window

* `Group Window` 是使用window（w: GroupWindow）子句定义的，并且**必须由as子句指定一个别名**

* 为了按窗口对表进行分组，窗口的别名必须在group by 子句中，像常规的分组字段一样引用

  ```java
  Table table = input
    .window([w:GroupWindow] as "w")	// 定义窗口，别名为w
    .groupBy("w, a")								// 按照字段 a 和窗口 w 分组
    .select("a, b.sum");						// 聚合
  ```

* Table API 提供了一组具有特定语义的预定于window类，这些类会被转换为底层DataStream 或 DataSet 的窗口操作
* 分组窗口分为三种：
  * 滚动窗口
  * 滑动窗口
  * 会话窗口

##### 9.4.1.1 滚动窗口（Tumbling windows）

* 滚动窗口（Tumbling windows）要用Tumble类来定义

  ```java
  // Tumbling Event-time Window （事件时间字段rowtime）
  .window(Tumble.over("10.minutes").on("rowtime").as("w"))
    
  // Tumbling Process-time Window （处理时间字段proctime）
  .window(Tumble.over("10.minutes").on("proctime").as("w"))
    
  // Tumbling Row-count Window （类似于计数窗口，按处理时间排序，10行一组）
  .window(Tumble.over("10.rows").on("proctime").as("w"))
  ```

  * over: 定义窗口长度
  * on: 用来分组（按时间间隔）或者排序（按行数）的时间字段
  * as: 别名，必须出现在后面的groupBy中

##### 9.4.1.2 滑动窗口（Sliding windows）

* 滑动窗口（Sliding windows）要用Slide类来定义

```java
// Sliding Event-time Window
.window(Slide.over("10.minutes").every("5.minutes").on("rowtime").as("w"))
  
// Slding Processing-time Window
.window(Slide.over("10.minutes").every("5.minutes").on("proctime").as("w"))
  
// Sliding Row-count window
.window(Slding.over("10.rows").every("5.rows").on("proctime").as("w"))
```

- over：定义窗口长度
- every：定义滑动步长
- on：用来分组（按时间间隔）或者排序（按行数）的时间字段
- as：别名，必须出现在后面的groupBy中

##### 9.4.1.3 会话窗口（Session windows）

* 会话窗口（Session windows）要用Session类来定义

  ```java
  // Session Event-time Window
  .window(Session.withGap("10.minutes").on("rowtime").as("w"))
    
  // Session Processing-time Window
  .window(Session.withGap("10.minutes").on("proctime").as("w"))
  ```

* withGap：会话时间间隔

* on：用来分组（按时间间隔）或者排序（按行数）的时间字段

* as：别名，必须出现在后面的groupBy中

#### 9.4.2 SQL中的Group Windows

Group Window定义在SQL查询的Group By子句中

* TUMBLE(time_attr, interval)
  * 定义一个滚动窗口，第一个参数是时间字段，第二个参数是窗口长度
* HOP(time_attr, interval, interval)
  * 定义一个滑动窗口，第一个参数是时间字段，**第二个参数是窗口滑动步长， 第三个参数是窗口长度**
* SESSION(time_attr, interval)
  * 定义一个会话窗口，第一个参数是时间字段，第二个参数是窗口间隔

#### 9.4.3 Over Windows

* **Over window 聚合是标准SQL中已有的（over子句），可以在查询的SELECT子句中定义**

* Over window聚合，会**针对每个输入行**，计算相邻行范围内的聚合（但因为数据是来一个处理一个，所以只能知道之前的数据，所以暂时不支持聚合该数据之后的数据）

* Over windows使用window（w:overwindows）子句定义，并在select()方法中通过**别名**来引用

  ```java
  Table table = input
    .window([w: OverWindow] as "w")
    .select("a, b.sum over w, c.min over w");
  ```

* Table API 提供了Over类，来配置Over窗口的属性

##### 无界Over Windows

* 可以在时间时间或处理时间，以及指定为时间间隔、或行计数的范围内，定义 Over windows

* 无界的 Over Window 是使用常量指定的

  ```java
  // 无界的事件时间over window（时间字段 "rowtime"）
  .window(Over.partitionBy("a").orderBy("rowtime").preceding(UNBOUNDED_RANGE).as("w"))
    
  // 无界的处理时间over window（时间字段 "proctime"）
  .window(Over.partitionBy("a").orderBy("proctime").preceding(UNBOUNDED_RANGE).as("w"))
  
  // 无界的事件时间row-count over window（时间字段 "rowtime"）
  .window(Over.partitionBy("a").orderBy("rowtime").preceding(UNBOUNDED_ROW).as("w"))
  
  // 无界的处理时间row-count over window（时间字段 "proctime"）
  .window(Over.partitionBy("a").orderBy("proctime").preceding(UNBOUNDED_ROW).as("w"))
  ```

  *partitionBy是可选项*

##### 有界Over Windows

* 有界的over window是用间隔的大小指定的

  ```java
  // 有界的事件时间over window (时间字段 "rowtime"，之前1分钟)
  .window(Over.partitionBy("a").orderBy("rowtime").preceding("1.minutes").as("w"))
  
  // 有界的处理时间over window (时间字段 "rowtime"，之前1分钟)
  .window(Over.partitionBy("a").orderBy("porctime").preceding("1.minutes").as("w"))
  
  // 有界的事件时间Row-count over window (时间字段 "rowtime"，之前10行)
  .window(Over.partitionBy("a").orderBy("rowtime").preceding("10.rows").as("w"))
  
  // 有界的处理时间Row-count over window (时间字段 "rowtime"，之前10行)
  .window(Over.partitionBy("a").orderBy("proctime").preceding("10.rows").as("w"))
  ```

#### 9.4.4 SQL 中的 Over Windows

* 用Over做窗口聚合时，所有聚合必须在同一窗口上定义，也就是说必须是相同的分区、排序和范围
* 目前仅支持在当前行范围之前的窗口
* ORDER BY 必须在单一的时间属性上指定

```java
SELECT COUNT(amount) OVER (
	PARTTION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM Orders
  
// 也可以做多个聚合
SELECT COUNT(amount) OVER w, SUM(amount) OVER w
FROM Orders
WINDOW w AS (
	PARTITION BY user
	ORDER BY proctime
	ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

#### 9.4.5 测试代码

```java
com.sqlist.apitest.tablesql.TableSqlTest5Window
```

## 10 函数（Functions）

![image-20210712170714507](https://ww1.sinaimg.cn/large/6af0fe46ly1gsea1hdmogj20jn0bmaa6.jpg)

![image-20210712170729703](https://ww1.sinaimg.cn/large/6af0fe46ly1gsea1q6yo1j20hk0ast8r.jpg)

### 10.1 用户自定义函数（UDF）

- 用户定义函数（User-defined Functions，UDF）是一个重要的特性，它们显著地扩展了查询的表达能力

  *一些系统内置函数无法解决的需求，我们可以用UDF来自定义实现*

- **在大多数情况下，用户定义的函数必须先注册，然后才能在查询中使用**

- 函数通过调用 `registerFunction()` 方法在 TableEnvironment 中注册。当用户定义的函数被注册时，它被插入到 TableEnvironment 的函数目录中，这样Table API 或 SQL 解析器就可以识别并正确地解释它

#### 10.1.1 标量函数（Scalar Functions）

* `Scalar Function` 类似于 map，一对一

* 用户定义的标量函数，可以将**0、1或多个标量值，映射到新的标量值**

* 为了定义标量函数，必须在 org.apache.flink.table.functions 中扩展基类Scalar Function，并实现（一个或多个）求值（eval）方法

* **！！！ 标量函数的行为由求值方法决定，求值方法必须public公开声明并命名为 eval**

  ```java
  public static class HashCode extends ScalarFunction {
    private int factor = 13;
  
    public HashCode(int factor) {
      this.factor = factor;
    }
  
    public int eval(String id) {
      return id.hashCode() * 13;
    }
  }
  ```

##### 测试代码

```java
com.sqlist.apitest.tablesql.udf.UdfTest1ScalarFunction
```

#### 10.1.2 表格函数（Table Functions）

* `Table Function`类似与flatMap，一对多

* 用户定义的表函数，也可以将**0、1或多个标量值作为输入参数；与标量函数不同的是，它可以返回任意数量的行作为输出，而不是单个值**

* 为了定义一个表函数，必须扩展 org.apache.flink.table.functions 中的基类 TableFunction 并实现（一个或多个）求值方法

* **表函数的行为由其求值方法决定，求值方法必须是 public 的，并命名为 eval**

  ```java
  public static class Split extends TableFunction<Tuple2<String, Integer>> {
    private String separator = ",";
  
    public Split(String separator) {
      this.separator = separator;
    }
  
    public void eval(String str) {
      for (String s : str.split(separator)) {
        collect(new Tuple2<>(s, s.length()));
      }
    }
  }
  ```

##### 测试代码

```java
com.sqlist.apitest.tablesql.udf.UdfTest2TableFunction
```

#### 10.1.3 聚合函数（Aggregation Functions）

* **聚合，多对一，类似前面的窗口聚合**
* 用户自定义聚合函数（User-Defined Aggregate Functions, UDAGGs）可以把一个表中的数据，聚合成一个标量值
* 用户定义的聚合函数，是通过继承AggregateFunction抽象类实现的
  ![image-20210713155809679](https://ww1.sinaimg.cn/large/6af0fe46ly1gsfdp3bq4nj20hh08t3ym.jpg)
* AggregationFunction要求必须实现的方法
  * `createAccumulator()`
  * `accumulate()`
  * `getValue()`
* AggregateFunction的工作原理如下：
  * 首先，需要一个累加器（Accumulator），用来保存聚合中间结果的数据结构；可以通过调用`createAccumulator()`方法创建空累加器
  * 随后，对每个输入行调用函数的`accumulate()`方法来更新累加器
  * 处理完所有行后，将调用函数的`getValue()`方法来计算并返回最终结果

##### 测试代码

```java
com.sqlist.apitest.tablesql.udf.UdfTest3AggregateFunction
```

#### 10.1.4 表聚合函数

* **聚合，多对多**
* 用户定义的表聚合函数（User-Defined Table Aggregate Functions，UDTAGGs），可以把一个表中数据，聚合为具有多行和多列的结果表
* 用户定义表聚合函数，是通过继承`TableAggregateFunction`抽象类来实现
  ![image-20210714161353681](https://ww1.sinaimg.cn/large/6af0fe46ly1gsgkwyy43lj20h7094dfy.jpg)

* `AggregationFunction`要求必须实现的方法：
  * `createAccumulator()`
  * `accumulate()`
  * `emitValue()`
* `TableAggregateFunction`的工作原理如下：
  * 首先，它同样需要一个累加器（Accumulator），它是保存聚合中间结果的数据结构。通过调用 `createAccumulator()` 方法可以创建空累加器。
  * 随后，对每个输入行调用函数的 `accumulate()` 方法来更新累加器。
  * 处理完所有行后，将调用函数的 `emitValue()` 方法来计算并返回最终结果。

##### 测试代码

```java
com.sqlist.apitest.tablesql.udf.UdfTest4TableAggregateFunction
```































