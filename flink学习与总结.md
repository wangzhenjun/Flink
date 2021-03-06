
# 第一章  Flink核心概念综述
## 一、Flink 简介

Apache Flink 诞生于柏林工业大学的一个研究性项目，原名 StratoSphere 。
- 2014 年，由 StratoSphere 项目孵化出 Flink，并于同年捐赠 Apache，之后成为 Apache 的顶级项目。
- 2019 年 1 年，阿里巴巴收购了 Flink 的母公司 Data Artisans，并宣布开源内部的 Blink，Blink 是阿里巴巴基于 Flink 优化后的版本，增加了大量的新功能，并在性能和稳定性上进行了各种优化，经历过阿里内部多种复杂业务的挑战和检验。同时阿里巴巴也表示会逐步将这些新功能和特性 Merge 回社区版本的 Flink 中，因此 Flink 成为目前最为火热的大数据处理框架。

简单来说，Flink 是一个分布式的流处理框架，它能够对有界和无界的数据流进行高效的处理。Flink 的核心是流处理，当然它也能支持批处理，Flink 将批处理看成是流处理的一种特殊情况，即数据流是有明确界限的。这和 Spark Streaming 的思想是完全相反的，Spark Streaming 的核心是批处理，它将流处理看成是批处理的一种特殊情况， 即把数据流进行极小粒度的拆分，拆分为多个微批处理。

Flink 有界数据流和无界数据流：










Spark Streaming 数据流的拆分：

![3ff7ce0aed2a41146ac0fa719bf867c8.png](en-resource://database/999:1)





## 二、Flink 核心架构

Flink 采用分层的架构设计，从而保证各层在功能和职责上的清晰。如下图所示，由上而下分别是 API & Libraries 层、Runtime 核心层以及物理部署层：


![675007bf4d7dc2a18bf0c3f19a3284e9.png](en-resource://database/1003:1)





### 2.1 API & Libraries 层

这一层主要提供了编程 API 和 顶层类库：

+ 编程 API : 用于进行流处理的 DataStream API 和用于进行批处理的 DataSet API；
+ 顶层类库：包括用于复杂事件处理的 CEP 库；用于结构化数据查询的 SQL & Table 库，以及基于批处理的机器学习库 FlinkML 和 图形处理库 Gelly。

### 2.2 Runtime 核心层

这一层是 Flink 分布式计算框架的核心实现层，包括作业转换，任务调度，资源分配，任务执行等功能，基于这一层的实现，可以在流式引擎下同时运行流处理程序和批处理程序。

### 2.3 物理部署层

Flink 的物理部署层，用于支持在不同平台上部署运行 Flink 应用。

## 三、Flink 分层 API

在上面介绍的 API & Libraries 这一层，Flink 又进行了更为具体的划分。具体如下：

![89b057bc668f35e3a830234b7667f43f.png](en-resource://database/1005:1)


按照如上的层次结构，API 的一致性由下至上依次递增，接口的表现能力由下至上依次递减，各层的核心功能如下：

### 3.1 SQL & Table API

SQL & Table API 同时适用于批处理和流处理，这意味着你可以对有界数据流和无界数据流以相同的语义进行查询，并产生相同的结果。除了基本查询外， 它还支持自定义的标量函数，聚合函数以及表值函数，可以满足多样化的查询需求。 

### 3.2 DataStream & DataSet API

DataStream &  DataSet API 是 Flink 数据处理的核心 API，支持使用 Java 语言或 Scala 语言进行调用，提供了数据读取，数据转换和数据输出等一系列常用操作的封装。

### 3.3 Stateful Stream Processing

Stateful Stream Processing 是最低级别的抽象，它通过 Process Function 函数内嵌到 DataStream API 中。 Process Function 是 Flink 提供的最底层 API，具有最大的灵活性，允许开发者对于时间和状态进行细粒度的控制。

## 四、Flink 集群架构

### 4.1  核心组件

按照上面的介绍，Flink 核心架构的第二层是 Runtime 层， 该层采用标准的 Master - Slave 结构， 其中，Master 部分又包含了三个核心组件：Dispatcher、ResourceManager 和 JobManager，而 Slave 则主要是 TaskManager 进程。它们的功能分别如下：

- **JobManagers** (也称为 *masters*) ：JobManagers 接收由 Dispatcher 传递过来的执行程序，该执行程序包含了作业图 (JobGraph)，逻辑数据流图 (logical dataflow graph) 及其所有的 classes 文件以及第三方类库 (libraries) 等等 。紧接着 JobManagers 会将 JobGraph 转换为执行图 (ExecutionGraph)，然后向 ResourceManager 申请资源来执行该任务，一旦申请到资源，就将执行图分发给对应的 TaskManagers 。因此每个作业 (Job) 至少有一个 JobManager；高可用部署下可以有多个 JobManagers，其中一个作为 *leader*，其余的则处于 *standby* 状态。
- **TaskManagers** (也称为 *workers*) : TaskManagers 负责实际的子任务 (subtasks) 的执行，每个 TaskManagers 都拥有一定数量的 slots。Slot 是一组固定大小的资源的合集 (如计算能力，存储空间)。TaskManagers 启动后，会将其所拥有的 slots 注册到 ResourceManager 上，由 ResourceManager 进行统一管理。
- **Dispatcher**：负责接收客户端提交的执行程序，并传递给 JobManager 。除此之外，它还提供了一个 WEB UI 界面，用于监控作业的执行情况。
- **ResourceManager** ：负责管理 slots 并协调集群资源。ResourceManager 接收来自 JobManager 的资源请求，并将存在空闲 slots 的 TaskManagers 分配给 JobManager 执行任务。Flink 基于不同的部署平台，如 YARN , Mesos，K8s 等提供了不同的资源管理器，当 TaskManagers 没有足够的 slots 来执行任务时，它会向第三方平台发起会话来请求额外的资源。
![c9238c698878974166800063f033dac5.png](en-resource://database/1007:1)



### 4.2  Task & SubTask

上面我们提到：TaskManagers 实际执行的是 SubTask，而不是 Task，这里解释一下两者的区别：

在执行分布式计算时，Flink 将可以链接的操作 (operators) 链接到一起，这就是 Task。之所以这样做， 是为了减少线程间切换和缓冲而导致的开销，在降低延迟的同时可以提高整体的吞吐量。 但不是所有的 operator 都可以被链接，如下 keyBy 等操作会导致网络 shuffle 和重分区，因此其就不能被链接，只能被单独作为一个 Task。  简单来说，一个 Task 就是一个可以链接的最小的操作链 (Operator Chains) 。如下图，source 和 map 算子被链接到一块，因此整个作业就只有三个 Task：

![4ec4a546065fff24a58ec77b403efed5.png](en-resource://database/1023:1)



解释完 Task ，我们在解释一下什么是 SubTask，其准确的翻译是： *A subtask is one parallel slice of a task*，即一个 Task 可以按照其并行度拆分为多个 SubTask。如上图，source & map 具有两个并行度，KeyBy 具有两个并行度，Sink 具有一个并行度，因此整个虽然只有 3 个 Task，但是却有 5 个 SubTask。Jobmanager 负责定义和拆分这些 SubTask，并将其交给 Taskmanagers 来执行，每个 SubTask 都是一个单独的线程。

### 4.3  资源管理

理解了 SubTasks ，我们再来看看其与 Slots 的对应情况。一种可能的分配情况如下：


![87925cae4bbd1f49b15e49668ff8084f.png](en-resource://database/1021:1)





这时每个 SubTask 线程运行在一个独立的 TaskSlot， 它们共享所属的 TaskManager 进程的TCP 连接（通过多路复用技术）和心跳信息 (heartbeat messages)，从而可以降低整体的性能开销。此时看似是最好的情况，但是每个操作需要的资源都是不尽相同的，这里假设该作业 keyBy 操作所需资源的数量比 Sink 多很多 ，那么此时 Sink 所在 Slot 的资源就没有得到有效的利用。

基于这个原因，Flink 允许多个 subtasks 共享 slots，即使它们是不同 tasks 的 subtasks，但只要它们来自同一个 Job 就可以。假设上面 souce & map 和 keyBy 的并行度调整为 6，而 Slot 的数量不变，此时情况如下：

![42a2761f53dc1ca59ff717e210de4d7a.png](en-resource://database/1025:1)


可以看到一个 Task Slot 中运行了多个 SubTask 子任务，此时每个子任务仍然在一个独立的线程中执行，只不过共享一组 Sot 资源而已。那么 Flink 到底如何确定一个 Job 至少需要多少个 Slot 呢？Flink 对于这个问题的处理很简单，默认情况一个 Job 所需要的 Slot 的数量就等于其 Operation 操作的最高并行度。如下， A，B，D 操作的并行度为 4，而 C，E 操作的并行度为 2，那么此时整个 Job 就需要至少四个 Slots 来完成。通过这个机制，Flink 就可以不必去关心一个 Job 到底会被拆分为多少个 Tasks 和 SubTasks。

![300096e0caebc5e6b49d0b65d9bac4b4.png](en-resource://database/1019:1)


 

### 4.4 组件通讯

Flink 的所有组件都基于 Actor System 来进行通讯。Actor system是多种角色的 actor 的容器，它提供调度，配置，日志记录等多种服务，并包含一个可以启动所有 actor 的线程池，如果 actor 是本地的，则消息通过共享内存进行共享，但如果 actor 是远程的，则通过 RPC 的调用来传递消息。
![6c6a73e2527c767cbaafff5ad3e6d56d.png](en-resource://database/1017:1)






## 五、Flink 的特点

最后基于上面的介绍，来总结一下 Flink 的特点：

+ Flink 是基于事件驱动 (Event-driven) 的应用，能够同时支持流处理和批处理；
+ 基于内存的计算，能够保证高吞吐和低延迟，具有优越的性能表现；
+ 支持精确一次 (Exactly-once) 语意，能够完美地保证一致性和正确性；
+ 分层 API ，能够满足各个层次的开发需求；
+ 支持高可用配置，支持保存点机制，能够提供安全性和稳定性上的保证；
+ 多样化的部署方式，支持本地，远端，云端等多种部署方案；
+ 具有横向扩展架构，能够按照用户的需求进行动态扩容；
+ 活跃度极高的社区和完善的生态圈的支持。

## 六. Flink相关内容总结

![3c1b6416caeb95d7a33aa7a827e2213f.png](en-resource://database/1027:1)


# 第二章 Flink核心算子
-----------------------------
![dca602348ec0c28e9dce8edb39d879b9.png](en-resource://database/1053:1)


## 1. DataStream Transformations

### 1.1 Map [DataStream → DataStream] 

对一个 DataStream 中的每个元素都执行特定的转换操作：
```java

DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
    });
```
测试样例
```java
DataStream<Integer> integerDataStream = env.fromElements(1, 2, 3, 4, 5);
integerDataStream.map((MapFunction<Integer, Object>) value -> value * 2).print();
// 输出 2,4,6,8,10
```

### 1.2 FlatMap [DataStream → DataStream]

FlatMap 与 Map 类似，但是 FlatMap 中的一个输入元素可以被映射成一个或者多个输出元素，示例如下：
```java

dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
 });
```
测试样例：
```java
String string01 = "one one one two two";
String string02 = "third third third four";
DataStream<String> stringDataStream = env.fromElements(string01, string02);
stringDataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out) throws Exception {
        for (String s : value.split(" ")) {
            out.collect(s);
        }
    }
}).print();
// 输出每一个独立的单词，为节省排版，这里去掉换行，后文亦同
one one one two two third third third four
```

### 1.3 Filter [DataStream → DataStream]

用于过滤符合条件的数据：

```java
env.fromElements(1, 2, 3, 4, 5).filter(x -> x > 3).print();
```

### 1.4 KeyBy 和 Reduce

- **KeyBy [DataStream → KeyedStream]** ：用于将相同 Key 值的数据分到相同的分区中；
- **Reduce [KeyedStream → DataStream]** ：用于对数据执行归约计算。
```java
//KeyBy
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple

//Reduce
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
  });
```
如下例子将数据按照 key 值分区后，滚动进行求和计算：

```java
DataStream<Tuple2<String, Integer>> tuple2DataStream = env.fromElements(new Tuple2<>("a", 1),
                                                                        new Tuple2<>("a", 2), 
                                                                        new Tuple2<>("b", 3), 
                                                                        new Tuple2<>("b", 5));
KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = tuple2DataStream.keyBy(0);

keyedStream.reduce((ReduceFunction<Tuple2<String, Integer>>) (value1, value2) ->
                   new Tuple2<>(value1.f0, value1.f1 + value2.f1)).print();

// 持续进行求和计算，输出：
(a,1)
(a,3)
(b,3)
(b,8)
```

KeyBy 操作存在以下两个限制：

- KeyBy 操作用于用户自定义的 POJOs 类型时，该自定义类型必须重写 hashCode 方法；
- KeyBy 操作不能用于数组类型。

### 1.5 Fold[KeyedStream → DataStream]

具有初始值的键控数据流上的“滚动”折叠。将当前元素与最后一个折叠值组合并发出新值。
当作用于序列(1,2,3,4,5)时，发出序列“start-1”、“start-1-2”、“start-1-2”、“start-1- 3”、…
```java
DataStream<String> result =
  keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
 });
```
### 1.6 Aggregations [KeyedStream → DataStream]

Aggregations 是官方提供的聚合算子，封装了常用的聚合操作，如上利用 Reduce 进行求和的操作也可以利用 Aggregations 中的 sum 算子重写为下面的形式：
```java

keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
```


```java
tuple2DataStream.keyBy(0).sum(1).print();
```

除了 sum 外，Flink 还提供了 min , max , minBy，maxBy 等常用聚合算子：

```java
// 滚动计算指定key的最小值，可以通过index或者fieldName来指定key
keyedStream.min(0);
keyedStream.min("key");
// 滚动计算指定key的最大值
keyedStream.max(0);
keyedStream.max("key");
// 滚动计算指定key的最小值，并返回其对应的元素
keyedStream.minBy(0);
keyedStream.minBy("key");
// 滚动计算指定key的最大值，并返回其对应的元素
keyedStream.maxBy(0);
keyedStream.maxBy("key");

```
### 1.7 Window [KeyedStream → WindowedStream]

可以在已经分区的KeyedStreams上定义Windows。Windows根据一些特征(例如，最近5秒内到达的数据)对每个密钥中的数据进行分组。有关windows的完整描述，请参见windows。
```java

dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
```

### 1.8 Union [DataStream* → DataStream]

用于连接两个或者多个元素类型相同的 DataStream 。当然一个 DataStream 也可以与其本生进行连接，此时该 DataStream 中的每个元素都会被获取两次：

```java
DataStreamSource<Tuple2<String, Integer>> streamSource01 = env.fromElements(new Tuple2<>("a", 1), 
                                                                            new Tuple2<>("a", 2));
DataStreamSource<Tuple2<String, Integer>> streamSource02 = env.fromElements(new Tuple2<>("b", 1), 
                                                                            new Tuple2<>("b", 2));
streamSource01.union(streamSource02);
streamSource01.union(streamSource01,streamSource02);
```

### 1.9 Connect [DataStream,DataStream → ConnectedStreams]

Connect 操作用于连接两个或者多个类型不同的 DataStream ，其返回的类型是 ConnectedStreams ，此时被连接的多个 DataStreams 可以共享彼此之间的数据状态。但是需要注意的是由于不同 DataStream 之间的数据类型是不同的，如果想要进行后续的计算操作，还需要通过 CoMap 或 CoFlatMap 将 ConnectedStreams  转换回 DataStream：

```java
DataStreamSource<Tuple2<String, Integer>> streamSource01 = env.fromElements(new Tuple2<>("a", 3), 
                                                                            new Tuple2<>("b", 5));
DataStreamSource<Integer> streamSource02 = env.fromElements(2, 3, 9);
// 使用connect进行连接
ConnectedStreams<Tuple2<String, Integer>, Integer> connect = streamSource01.connect(streamSource02);
connect.map(new CoMapFunction<Tuple2<String, Integer>, Integer, Integer>() {
    @Override
    public Integer map1(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }

    @Override
    public Integer map2(Integer value) throws Exception {
        return value;
    }
}).map(x -> x * 100).print();

// 输出：
300 500 200 900 300
```

### 1.10 Split 和 Select

- **Split [DataStream → SplitStream]**：用于将一个 DataStream 按照指定规则进行拆分为多个 DataStream，需要注意的是这里进行的是逻辑拆分，即 Split 只是将数据贴上不同的类型标签，但最终返回的仍然只是一个 SplitStream；
- **Select [SplitStream → DataStream]**：想要从逻辑拆分的 SplitStream 中获取真实的不同类型的 DataStream，需要使用 Select 算子，示例如下：

```java
DataStreamSource<Integer> streamSource = env.fromElements(1, 2, 3, 4, 5, 6, 7, 8);
// 标记
SplitStream<Integer> split = streamSource.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        output.add(value % 2 == 0 ? "even" : "odd");
        return output;
    }
});
// 获取偶数数据集
split.select("even").print();
// 输出 2,4,6,8
```

### 1.11 project [DataStream → DataStream]

project 主要用于获取 tuples 中的指定字段集，示例如下：

```java
DataStreamSource<Tuple3<String, Integer, String>> streamSource = env.fromElements(
                                                                         new Tuple3<>("li", 22, "2018-09-23"),
                                                                         new Tuple3<>("ming", 33, "2020-09-23"));
streamSource.project(0,2).print();

// 输出
(li,2018-09-23)
(ming,2020-09-23)
```

# 第三章 生产环境Flink集群的开发应用

### 3.1 简介 
生产环境集群地址：http://cloud-in.gameyw.netease.com:8083/g-prog-ai/flink/v2/1/#/overview
- 工作台Dashboard
主界面会显示当前运行的任务和当天完成的任务（失败任务也会显示在此）
![0a490c82322c78fd5046d05c8638fdca.png](en-resource://database/1029:1)
如果你点击进入你启动的job中你将会得到一个视图，并且你可以检查单个操作，例如：看到处理元素的数量，收发记录数和并行度，状态等运行时的一些参数
![06c80ff206a47e16cc01aaaf030af1f2.png](en-resource://database/1031:1)

如果任务运行过程中失败，就可以在下面的视图查看异常日志。
![bbf87150162f37582e1a51e6b76f2c2a.png](en-resource://database/1037:1)


- 如何查看日志
通过K8S命令，每一次都拉取过去所有的日志信息，排查任务失败的原因。
```shell
/logs/workspace/k8s# kubectl -n g-prog-ai logs pod/flink-v2-g-prog-ai-taskmanager-694b8ff5b-bzdls --kubeconfig g-prog-ai_kubeconfig
```
![2ec08135d2bb1d7de1d520bc0108a1d6.png](en-resource://database/1039:1)

下面是通过k8s查看logs的一些相关命令
 kubectl logs --help


```shell
Print the logs for a container in a pod or specified resource. If the pod has only one container, the container name is
optional.


Aliases:ruby
logs, log


Examples:
  # Return snapshot logs from pod nginx with only one container 从pod nginx返回只有一个容器的快照日志
  kubectl logs nginx
  
  # Return snapshot logs for the pods defined by label app=nginx  返回标签app=nginx定义的pod的快照日志
  kubectl logs -lapp=nginx
  
  # Return snapshot of previous terminated ruby container logs from pod web-1  从pod web-1返回先前终止的容器日志的快照
  kubectl logs -p -c ruby web-1
  
  # Begin streaming the logs of the ruby container in pod web-1
  kubectl logs -f -c ruby web-1
  
  # Display only the most recent 20 lines of output in pod nginx  在pod nginx中只显示最近的20行输出
  kubectl logs --tail=20 nginx
  
  # Show all logs from pod nginx written in the last hour 显示所有来自pod nginx在过去一小时内写的日志
  kubectl logs --since=1h nginx
  
  # Return snapshot logs from first container of a job named hello
  kubectl logs job/hello
  
  # Return snapshot logs from container nginx-1 of a deployment named nginx
  kubectl logs deployment/nginx -c nginx-1


Options:
  -c, --container='': Print the logs of this container
  -f, --follow=false: Specify if the logs should be streamed.
      --limit-bytes=0: Maximum bytes of logs to return. Defaults to no limit.
      --pod-running-timeout=20s: The length of time (like 5s, 2m, or 3h, higher than zero) to wait until at least one pod is running
  -p, --previous=false: If true, print the logs for the previous instance of the container in a pod if it exists.
  -l, --selector='': Selector (label query) to filter on.
      --since=0s: Only return logs newer than a relative duration like 5s, 2m, or 3h. Defaults to all logs. Only one of
since-time / since may be used.
      --since-time='': Only return logs after a specific date (RFC3339). Defaults to all logs. Only one of since-time /
since may be used.
      --tail=-1: Lines of recent log file to display. Defaults to -1 with no selector, showing all log lines otherwise
10, if a selector is provided.
      --timestamps=false: Include timestamps on each line in the log output


Usage:
  kubectl logs [-f] [-p] (POD | TYPE/NAME) [-c CONTAINER] [options]


Use "kubectl options" for a list of global command-line options (applies to all commands).
```





-
### 3.2 开发
1.首先准备开发环境
- win安装java，scala，配置环境变量
- 下载最新版的flink tar包，解压到本地，然后start_cluster.dat，算是启动了本地版的flink
- 安装IDEA
（以上步骤相对基础，不再进一步阐述）

2.当IDE准备就绪后，开始创建一个项目名为FlinkJava的maven项目，如下图。
![111955f771df0869ec8191f0df55dd46.png](en-resource://database/1045:1)
![208c44f32e6cdeca71b9bc89ac255b9d.png](en-resource://database/1047:1)

3.在新窗口打开bbb项目时，IDEA会提示我们是否自动导包。选择自动导包，如下图。
![0bbb9d0a3c34fd46c2a8c8e8c96ed285.png](en-resource://database/1043:1)

4.对pom.xml配置文件进行修改，如下代码。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.netease</groupId>
    <artifactId>godlike_flink</artifactId>
    <version>1.0-SNAPSHOT</version>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>6</source>
                    <target>6</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-streaming-scala -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.12</artifactId>
            <version>1.8.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_2.11</artifactId>
            <version>1.8.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-filesystem_2.11</artifactId>
            <version>1.8.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.8.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.10_2.11</artifactId>
            <version>1.8.1</version>
            <scope> compile</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>0.10.2.0</version>
            <scope>compile</scope>
        </dependency>

        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongo-java-driver</artifactId>
            <version>3.10.1</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-scala -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-scala_2.11</artifactId>
            <version>1.8.1</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.9</version>
        </dependency>
    </dependencies>
</project>
```

5. 通过maven进行编译，打包
![5a9180869682ba88d1c2e009b39e5c91.png](en-resource://database/1049:1)

6. 下面是实现“读取kafka原始日志，然后写回到Hdfs，计算每个动态的PV值”样例程序代码
```java
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.java.tuple.Tuple2;

import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.connectors.fs.bucketing.BucketingSink;
import org.apache.flink.streaming.connectors.fs.bucketing.DateTimeBucketer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;

import java.text.SimpleDateFormat;
import java.time.*;
import java.util.Date;

import java.util.Properties;
import com.alibaba.fastjson.JSONObject;

public class KafkaHdfs_ExposurePV {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers","test01.brokers.canal.netease.com:9092");
        properties.setProperty("group.id","flink-kafka-test");

        /*
        * kafka消费配置起始位置*/
        FlinkKafkaConsumer010<String> myConsumer = new FlinkKafkaConsumer010<String>("opd_god_resys_client_log",new SimpleStringSchema(),properties);
        DataStream<String> datastream = env.addSource(myConsumer);

        DataStream<Tuple2<String, Integer>> CleanData = datastream.map(new MapFunction<String, Tuple2<String,JSONObject>>() {
            @Override
            public Tuple2<String,JSONObject> map(String value) throws Exception {
                JSONObject jsonObject= JSONObject.parseObject(value);
                String f = "null";
                if(jsonObject.containsKey("f")){
                    f = jsonObject.getString("f");
                }else {
                    f= "null";
                }
                //System.out.println(f);
                return Tuple2.of(f,jsonObject);
            }
        }).filter(new FilterFunction<Tuple2<String, JSONObject>>() {
            @Override
            public boolean filter(Tuple2<String, JSONObject> value) throws Exception {
                return value.f0.equals("card_exposure");
            }
        }).map(new MapFunction<Tuple2<String,JSONObject>,Tuple2<String,Integer>>() {
            public Tuple2<String,Integer> map(Tuple2<String, JSONObject> value) throws Exception {
                //f = value.f0;
                //String user_uid = value.f1.getString("user_uid").trim();
                String feed_id = "null";
                if(value.f1.containsKey("moment_id")) {
                    feed_id = value.f1.getString("moment_id").trim();
                }
                //String rec_time = value.f1.getString("rec_time");
                return Tuple2.of(feed_id,1);
            }
        }).keyBy(0).timeWindow(Time.seconds(5)).sum(1);

        DataStream ResultStream = CleanData.map(new MapFunction<Tuple2<String, Integer>, String>() {
            public String map(Tuple2<String, Integer> value) throws Exception {
                Date dNow = new Date( );
                SimpleDateFormat ft = new SimpleDateFormat ("yyyy-MM-dd hh:mm:ss");
                return ft.format(dNow)+"\t"+value.f0+"\t"+value.f1;
            }
        });

        BucketingSink sink = new BucketingSink("hdfs://neophdfs/home/workspace/public_prog_godlike/flink_sink_test/");
        sink.setBucketer(new DateTimeBucketer("yyyyMMdd-HH", ZoneId.of("Asia/Shanghai")));
        sink.setBatchSize(1024*512*128L);
        sink.setBatchRolloverInterval(10*60*1000L);

        ResultStream.addSink(sink);

        env.execute("KafkaHdfs_ExposurePV");
    }

}

```
编译，打包成jar，OK!
### 3.3 提交job 
- flink任务提交
若要提交新的任务，点击“Submit new Job” -- “Add New+”，上传编译好的jar包工程

![e6d13942a3ba109c4c2e7ab98a9ad265.png](en-resource://database/1033:1)
下面就是参数配置：
![69ca530e446c39b3d3a295c9e30886be.png](en-resource://database/1041:1)


