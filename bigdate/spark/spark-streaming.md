#目录

* [概述](\#t1)
* [一个简单的例子](\#t2)
* [基本概念](\#t3)
	* [相关依赖](\#t3-1)
	* [初始化StreamingContext](\#t3-2)
	* [离散流(DStream)](\#t3-3)
	* [输入流和接收器](\#t3-4)
	* [DStream中的转换](\#t3-5)
	* [DStream的输出操作](\#t3-6)
	* [DataFrame和SQL操作](\#t3-7)
	* MLlib操作
	* 缓存/持久化
	* 检查点
	* 部署程序
	* 监控程序
* 性能优化
	* 减少批处理时间
	* 设置合理的批处理间隔
	* 内存调优
* 容灾处理
* 0.9.1以下版本迁移至1.x指南
* 接下来学习什么

## <a NAME="t1">概述</a>
Spark Streaming是一个基于Spark核心API实现的易拓展、高效、可靠的实时流数据处理拓展组件。Spark的数据可以从多种数据源获取，如Kafka, Flume, Twitter, ZeroMQ, Kinesis或TCP sockets，并且可以通过使用map, reduce, join and window等高级函数进行复杂的算法处理，最终处理过的数据可以写入文件系统、数据库或者实时可视化报表。实际上，你还能在流处理的过程中使用Spark的机器学习和图处理算法。
![](http://spark.apache.org/docs/latest/img/streaming-arch.png)
从内部看，他的工作原理如下图。Spark Streaming接收输入数据流，并将这些数据分割成多个batch（这里直接用batch，感觉批次听着挺怪），然后再由Spark engine处理并生成最终的结果。
![](http://spark.apache.org/docs/latest/img/streaming-flow.png)
Spark Streaming提供了一个叫做离散流(DStream)的高级抽象，用来表示连续的流数据。DStreams既可以从Kafka、Flume、Kinesis这样源数据的输入流创建，也可以从高级操作中或者其他DStreams中产生。从内部看，DStreams用来表示一个RDD序列。
本指南可以告诉你如何使用DStreams来开发Spark Streaming程序。你可以使用Scala、Java或者Python编写程序，所有版本这里都会提供。
	注意：在Python中有一些API不同或者不可用，这些地方将会高亮标记。

## <a NAME="t2">一个简单的例子</a>
在我们详细的介绍如何编写你的Spark Streaming程序之前，我们先来看一个简单的Spark Streaming程序是什么样子的。假如我们打算统计一个从数据服务器的TCP socket获取的文本文件中的单词个数，我们需要按照下边来做。

首先，我们需要导入StreamingContext，他是所有流处理功能的入口。我们可以设置2个本地线程，并设置批处理间隔为1s。
```python
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

# 创建一个本地StreamingContext，线程数为2，批处理间隔为1s
sc = SparkContext("local[2]", "NetworkWordCount")
ssc = StreamingContext(sc, 1)
```
使用这个context，我们可以从TCP创建一个DStream，并指定hostname和port。
```python
lines = ssc.socketTextStream("localhost", 9999)
```
lines DStream表示的就是将从数据服务器接收的数据流，每条记录就是一行文本。接下来我们根据空格将行分隔成单次。
```python
words = lines.flatMap(lambda line: line.split(" "))
```
flatMap是一个一对多的DStream操作，它会通过把源数据中每条记录生成多条新记录来创建一个新的DStream。在这个例子中，每一行将会被分割成多个单次，单次流将会赋值给words这个DStream。接下来，我们将会统计这些单次个数。
```python
# 统计每个batch中的word数
pairs = words.map(lambda word: (word, 1))
wordCounts = pairs.reduceByKey(lambda x, y: x + y)

# 打印出每个DStream生成的RDD中前十个元素
wordCounts.pprint()
``` 
words DStream将会被映射（一对一的转换）成(word, 1)这样的键值对，这样就变为获取每个batch中word的频次了。最后，wordCounts.pprint()将会打印出一部分（前10个）每秒生成的记录数。

要注意的是当这些代码执行的时候，Spark Streaming仅仅设置了程序开始时将会执行的计算，并没真正的开始执行。为了在前边这些转换设置之后开始执行，我们需要调用：
```python
ssc.start()             # 开始执行
ssc.awaitTermination()  # 等待执行结束
```
完整的代码可以从参考Spark Streaming示例代码[NetworkWordCount](https://github.com/apache/spark/blob/master/examples/src/main/python/streaming/network_wordcount.py)。
如果你已经下载并构建了Spark，你可以按照如下方法运行示例。你首先需要运行Netcat（一个在多数类Unix系统上都有的小工具）作为数据服务器：
```shell
$ nc -lk 9999
```
这时，在不同的终端，你可以如下启动示例
```shell
$ ./bin/spark-submit examples/src/main/python/streaming/network_wordcount.py localhost 9999
```
Then, any lines typed in the terminal running the netcat server will be counted and printed on screen every second. It will look something like the following.
这时，在运行的netcat终端输入的每行数据每秒都会被计数并且打印在屏幕上，如下：
```shell
# TERMINAL 1:
# Running Netcat

$ nc -lk 9999

hello world
```
```shell
# TERMINAL 2: RUNNING network_wordcount.py

$ ./bin/spark-submit examples/src/main/python/streaming/network_wordcount.py localhost 9999
...
-------------------------------------------
Time: 2014-10-14 15:25:21
-------------------------------------------
(hello,1)
(world,1)
...

```

## <a NAME="t3">基本概念</a>
接下来，我们通过回顾之前那个简单的例子，来详细的说明Spark Streaming的基本概念。

#### <a NAME="t3-1">相关依赖</a>
与Spark相似，Spark Streaming的依赖包可以从Maven仓库获取。你需要把下边的依赖加入你的Maven或SBT项目，以开发你自己的程序。
```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.10</artifactId>
    <version>1.4.1</version>
</dependency>
```
为了从Kafka、Flume和Kinesis这样未包含在核心API中的数据源获取数据，你还需要将相应的模块spark-streaming-xyz_2.10加入到依赖中。例如，下边这些组件：
![](https://github.com/NotBadPad/translation/raw/master/img/spark-1.png)

若要获取最新的列表，请参考maven仓库中完整的支持的列表。

#### <a NAME="t3-2">初始化StreamingContext</a> ####
在初始化StreamingContext的时候，StreamingContext必须首先被创建，它是Spark Streaming所有功能的入口。
StreamingContext可以使用SparkContext来创建:
```python
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

sc = SparkContext(master, appName)
ssc = StreamingContext(sc, 1)
```
参数appName你的应用在集群UI中显示的名字，参数master是Spark,Mesos或YARN集群的URL，如果是本地模式则使用特殊字符串local\[\*\]。实际上，当在集群上运行的时候，我们并不会把master硬编码在程序里，而是通过spark-submit启动应用并接受它。然而，对于本地测试或单元测试，你可以直接通过local\[\*\]在同进程运行Spark Streaming（可以检测到本地系统的核数）。
批处理间隔的设置必须依据应用潜在的需求和集群可用的资源。详情可以参看调优部分。
context创建之后，我们需要做如下事情：

1. 通过定义输入DStreams来定义输入source
2. 通过转换和输出操作定义DStreams的流计算
3. 使用streamingContext.start()开始接收数据并处理
4. 使用streamingContext.awaitTermination()等待处理结束（手动或者由于错误）
5. 程序可以手动通过streamingContext.stop()结束

需要记住的点：

* 一旦context启动，不能再有新的流处理被设置或添加
* 一旦context停止，将不能重启
* 同一时刻一个JVM中只能有一个StreamingContext是活跃的
* StreamingContext的stop()也会结束SparkContext。如果仅仅要结束StreamingContext，需要将stop()的可选参数stopSparkContext设置为false
* SparkContext可以通过创建多个StreamingContext来重用，只要在下一个StreamingContext创建前前一个StreamingContext停止（SparkContext未停止）就行

#### <a NAME="t3-3">离散流(DStream)</a> ####
离散流或DStream是Spark Streaming提供的基本抽象。它表示一个连续的数据流，既可以是从source收到的输入数据流，也可以是通过转换输入流生成的。从内部看，DStream表示一组连续的RDD，RDD是Spark里不可变、分布式数据集的抽象。DStream中的每个RDD包含的数据都有一定的时间间隔，如下图所示：
![](http://spark.apache.org/docs/1.4.1/img/streaming-dstream.png)

任何DStream上的操作都转换为底层RDD上的操作。例如，之前将行流转为单词的例子中，flatMap操作在lines DStream中的每一个RDD上执行，生成words DStream的RDD，如下图：
![](http://spark.apache.org/docs/1.4.1/img/streaming-dstream-ops.png)

底层的RDD转换由Spark进行计算。DStream操作隐藏了大部分细节，为开发者提供了便利的高级接口。这些操作将在后边的章节讨论。

#### <a NAME="t3-4">输入DStreams和接收器</a> ####
输入DStreams是一个用来表示从source接收到数据的流。在前边的例子中，lines就表示一个从netcat服务器接收到的流数据的DStream。所有的输入DStreams（除了文件流，本节后边讨论）都与一个Receiver对象关联，它从source接收数据并存储在Spark的内存以便计算。
Spark Streaming提供了两类内置数据source：

* 基本source：这些源在StreamingContext API中直接有效。例如：文件系统、socket连接或Akka actors。
* 高级source：像Kafka, Flume, Kinesis, Twitter这样需要额外的工具类来使用。这些需要额外的依赖，在连接一节中已经讨论过。

我们将在接下来的章节中讨论每类中提到的source。
注意，如果你想在应用中接收多个并行的数据流，你可以创建多输入DStreams（将会在性能调优章节讨论）。这将会创建多个接收器并行的接收数据。但是要注意的是一个Spark worker/executor是一个持续运行的任务，因此它将占用从Spark Streaming申请的核中的一个。因此Spark Streaming应用需要申请足够的核接收数据，一遍运行接收器，这是很重要的。
需要记住的点：

* 当以本地模式运行时，不要使用local或local\[1\]作为master的URL。这意味着只有一个线程被用于执行本地任务。如果你使用一个基于接收器的输入DStream，这是仅有的线程将被用来运行接收器，将没有线程用来接收数据。因此，当已本地模式运行时，要使用local\[n\]作为URL，其中n>接收器的数量。
* 当拓展的逻辑运行在集群上时，从Spark Streaming应用申请的核的数量一定要大于接收器的数量，否则系统将能够接收数据，但是无法处理。

##### 基本source #####
我们已经看过前边那个简单的例子了，他创建了一个从TCP socket接收文本数据的DStream。除了sockets，StreamingContext API还提供了从文件和Akka actors作为输入创建DStreams的方法。

* File Streams：DStreams可以通过如下方式创建，能够从任何文件系统接收数据，兼容了HDFS API：
```python
 streamingContext.textFileStream(dataDirectory)
``` 	
Spark Streaming会监视dataDirectory目录并处理目录中创建的文件（写入嵌套目录的不支持）。要注意：

* 所有文件要拥有相同的格式
* 文件必须要通过移动或重命名子的方式在 dataDirectory中创建
* 文件一点移入，就不能改变，如果文件是持续写入的，新数据将不会别读取
**Python API：**fileStream在python中是不可用的，而textFileStream 是可用的

* 基于自定义Actors的Streams：DStreams可以通过使用streamingContext.actorStream(actorProps, actor-name)来接收Akka actors的数据创建。详情见自定义接收器指南。
**Python API：**actorStream 在python中是不可用的
* 以RDDs队列作为Streams：为了用测试数据测试 Spark Streaming程序，也可以使用streamingContext.queueStream(queueOfRDDs)创建一个基于RDDs队列的DStream。DStream会把push进队列的RDD作为批处理数据，像流那样处理它们。

想要获取更多细节，可以查看Python中StreamingContext相关的API文档。

##### 高级source #####
**Python API** Spark 1.4.1的source中，Python API只有Kafka是可用的，其他的我们将在将来加入。
这一类sources需要额外的库，其中有一些有复杂的依赖。因此为了减少依赖库之间的冲突，创建从source中DStreams的功能被封装在了单独的库里边，以便需要时轻易的被引用。例如你想从Twitter流的数据中创建DStreams，你可以按如下步骤：

* 链接：把包spark-streaming-twitter_2.10加入到SBT/Maven项目的依赖里边
* 编程：像下边一样导入TwitterUtils，并使用TwitterUtils.createStream创建DStreams
* 部署：将所有依赖打成jar包，然后部署。之后部署章节将会详细讲述。
```java
import org.apache.spark.streaming.twitter.*;

TwitterUtils.createStream(jssc);
```
注意在Spark shell中这类sources是不可用的。因此依赖于高级source的程序不能再shell下测试。如果你真的想要使用，需要把所有maven中依赖的jar包都下载下来加入到classpath。
其中一些高级source如下：

* Kafka：Spark Streaming 1.4.1与Kafka 0.8.2.1兼容
* Flume：Spark Streaming 1.4.1与Flume 1.4.0兼容
* Kinesis：参考Kinesis集成指南
* Twitter: 懒得啰嗦，用Twitter的话估计也用不着看中文了

##### 自定义source #####
**Python API：**同样不支持python
输入DStreams也可以从自定义source创建，你需要做的是定义一个receiver从自定义source接收数据，并推送到Spark。

##### Receiver的可靠性 #####
有两种方式保证其可靠性。Sources（像Kafka和Flume）允许转换数据被确认。如果系统从可靠source接收的数据被确认是正确的，就可以任务没有数据因为任何错误而丢失。这导致有两种receivers：

* 可靠地Receiver：可靠的receiver会在收到数据并存储到Spark后向source发送确认。
* 不可靠的Receiver：不可靠的Receiver不会像source发送确认，这种可以用于不支持确认的source，而且对于一些可靠source并不希望或者需要确认这种复杂的处理。
更多关系怎样写一个可靠的receiver参考自定义Receiver指南。

#### <a NAME="t3-5">DStreams中的转换</a> ####
与RDD相似，转换允许DStream中的数据被修改。DStream支持很多普通Spark RDD的转换操作。其中比较常用的如下：

* 
	* map(func)：将原DStream中的每一项通过func处理后，返回一个新的DStream
	* flatMap(func)：与map相似，每一个输入项可以被映射成0个或多个项
	* filter(func)：返回通过func过滤后的项
	* repartition(numPartitions)：通过创建更多或更少的partition来改变DStream的并行性
	* union(otherStream)：将原DStream和otherStream联接后作为一个新的DStream返回
	* count()：通过统计每个RDD元素的数量，最后返回一个只有个单值RDD的DStream
	* reduce(func)：通过使用func（两个参数、一个返回值）进行聚合操作，返回一个单值的DStream。这个func应该可以被组合，以便并行计算
	* countByValue()：如果DStream的元素类型是K，则返回一个(K,Long)类型的DStream，值是每个K在RDD中出现的频次
	* reduceByKey(func, [numTasks])：对于(K,V)类型的DStream，将会返回一个(K, V)的键值对，其中会把每个K对应的value通过func进行聚合。**注意**：默认情况下，将会使用默认参数设置并发任务（本地模式是2，集群模式下取决于设置的配置参数spark.default.parallelism）去进行group，你可以通过设置numTasks参数改变task的数量
	* join(otherStream, [numTasks])：对于两个(K, V)和(K, W)两个DStream，将会返回(K,(V,W))类型的DStream
	* cogroup(otherStream, [numTasks])：对于两个(K, V)和(K, W)两个DStream，将会返回(K,seq[V],seq[W])类型的DStream
	* transform(func)：通过使用RDD对RDD的func返回一个新的DStream，这可以被用于任意的DStream转换操作
	* updateStateByKey(func)：使用给定的func根据新值跟新对应key之前的状态，最终返回一个新状态的DStream。这可以用于维护key的任意状态。

这里边一些转换是值得详细讨论的

##### UpdateStateByKey操作 #####
updateStateByKey操作允许你维持任意的状态并同时用新的信息更新状态。你需要按如下步骤使用它：

* 定义状态，可以使任意数据类型
* 定义状态更新方法，定义一个能够使用之前状态和输入流中新值来更新状态的方法

我们来举个例子，假如你想统一个文本流中单词的变化的数量，这里变化数量就是integer的状态，我们定义如下：

```python
def updateFunction(newValues, runningCount):
    if runningCount is None:
       runningCount = 0
    return sum(newValues, runningCount)  # add the new values with the previous running count to get the new count
```
可以用于一个包含单词的DStream（包含(word,1)键值对的DStream）

```python
runningCounts = pairs.updateStateByKey(updateFunction)
```
每个单词都将会调用这个更新方法，使用1的序列作为newValues（来自(word,1)）,之前的count值作为runningCount。要看完整的python代码，参考[stateful_network_wordcount.py](https://github.com/apache/spark/blob/master/examples/src/main/python/streaming/stateful_network_wordcount.py)

注意使用updateStateByKey要求checkpoint目录被配置。

##### Transform操作 #####

transform操作允许在DStream上使用任意的RDD-to-RDD的方法，它可以被用来使用DStream API中为被暴露出来的RDD操作。例如，将数据流中每个批处理与另外一个数据集做join操作，DStream API并没有直接支持，但是你可以使用transform做到，这能够实现非常强大的拓展。例如，可以做实时数据清洗通过把输入数据和预先计算好的垃圾邮件数据作对比并过滤。

```python
spamInfoRDD = sc.pickleFile(...) # RDD containing spam information

# join data stream with spam information to do data cleaning
cleanedDStream = wordCounts.transform(lambda rdd: rdd.join(spamInfoRDD).filter(...))
```
注意提供的方法会在批处理间隔被调用，这允许你做时变操作，也就是说，RDD操作、分区数量、广播变量可以在批处理之间被改变。

##### Window操作 #####
Spark Streaming同样提供了窗口计算,允许你在数据的滑动窗口上使用转换。下图展示了一个滑动窗口：
![](http://spark.apache.org/docs/1.4.1/img/streaming-dstream-window.png)

如图所示，每次窗口在source DStream上滑动，window内的source RDD会合并，并且作为一个windowed DStream处理。在上边这个例子里，操作被用于最后三个时间单元的数据，并且每次滑动2个时间单元，这意味着每个window操作都需要2个参数：

* window length：window的持续时间
* sliding interval：window执行的间隔

这两个参数必须是source DStream中批处理间隔的倍数
我们用个例子来说明，如在前边例子中你想要统计30s内单词的数量，每隔10s执行一次。为达到目的，我们需要在过期30s的数据上使用reduceByKey操作处理(word,1)，这个操作使用reduceByKeyAndWindow就行：
```python
# Reduce last 30 seconds of data, every 10 seconds
windowedWordCounts = pairs.reduceByKeyAndWindow(lambda x, y: x + y, lambda x, y: x - y, 30, 10)
```
一些常用的window操作如下：

* 
	* window(windowLength, slideInterval)：返回一个给予window批处理计算的DStream
	* countByWindow(windowLength, slideInterval)：返回滑动窗口中元素的数量
	* reduceByWindow(func, windowLength, slideInterval)：返回一个单值的stream，通过在滑动间隔调用func来聚合。该方法需要能够关联，以便能够进行并行处理。
	* reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])：当在一个(K,V)类型的DStream调用时，会返回一个(K,V)类型的DStream，其中窗口中每个key的value值都会通过func聚合。**注意**：默认情况下，将会使用默认参数设置并发任务（本地模式是2，集群模式下取决于设置的配置参数spark.default.
	* reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])：A more efficient version of the above reduceByKeyAndWindow() where the reduce value of each window is calculated incrementally using the reduce values of the previous window. This is done by reducing the new data that enters the sliding window, and “inverse reducing” the old data that leaves the window. An example would be that of “adding” and “subtracting” counts of keys as the window slides. However, it is applicable only to “invertible reduce functions”, that is, those reduce functions which have a corresponding “inverse reduce” function (taken as parameter invFunc). Like in reduceByKeyAndWindow, the number of reduce tasks is configurable through an optional argument. Note that checkpointing must be enabled for using this operation.（直接看英文吧，这段实在翻译不动。这是reduceByKeyAndWindow的增强版，可以把之前窗口计算的值累加到当前窗口）
	* countByValueAndWindow(windowLength, slideInterval, [numTasks])：当在一个(K,V)类型的DStream调用时，会返回一个(K,Long)，value是每个key在窗口中出现的频次。

##### Join 操作 #####
最后，很值得介绍下在Spark Streaming中执行各类join操作有多容易。
###### Stream-stream joins ######
Streams很容易与其他Streams做连接。
```python
stream1 = ...
stream2 = ...
joinedStream = stream1.join(stream2)
```
在每个批处理间隔，stream1生成的RDD会与stream2生成的RDD做join，你也可以使用leftOuterJoin, rightOuterJoin, fullOuterJoin。此外，windows之间的join也很常用，这都很容易实现。
```python
windowedStream1 = stream1.window(20)
windowedStream2 = stream2.window(60)
joinedStream = windowedStream1.join(windowedStream2)
```
###### Stream-dataset joins ######
在之前解释DStream.transform的时候已经举过例子了，这里还有个一个window与数据集join的例子：
```python
dataset = ... # some RDD
windowedStream = stream.window(20)
joinedStream = windowedStream.transform(lambda rdd: rdd.join(dataset))
```
实际上你可以动态的改变加入的数据集。每个批处理间隔执行的transform可以使用当前dataset指向的数据集。
DStream完整的transformations操作列表可以参见API文档。

#### <a NAME="t3-6">DStream的输出操作</a> ####
输出操作允许DStream的数据被写入外部系统，如数据库或者文件系统。由于输出操作实际上允许转换后的数据被外部系统使用，这会触发DStream的转换操作实际的去执行。现在，我们看下下边这些输出操作的定义：

* print()：在运行streaming程序的驱动结点上打印每个批处理前10个元素。对于开发和调试很有用。**Python API** python中使用pprint()
* saveAsTextFiles(prefix, [suffix])：将DStream的内容保存到文本文件。文件名将使用 prefix 和 suffix:"prefix-TIME_IN_MS[.suffix]"
* saveAsObjectFiles(prefix, [suffix])：将DStream的内容保存为序列化的java对象文件，文件名将使用 prefix 和 suffix:"prefix-TIME_IN_MS[.suffix]"
* saveAsHadoopFiles(prefix, [suffix])：将DStream的内容保存为hadoop文件，文件名将使用 prefix 和 suffix:"prefix-TIME_IN_MS[.suffix]"
* foreachRDD(func)：最常用的操作是对stream生成都每个RDD使用func方法。这个方法会将每个RDD中的数据写入外部系统，一般是文件、网络或数据库。注意这个方法是在运行streaming程序的驱动进程中执行的，通常包含action操作以增强streaming RDD的计算。

##### 使用foreachRDD的设计模式 #####
dstream.foreachRDD是一个非常强大的将数据写入外部系统的原始方法。但是理解如何正确高效的使用该方法是很重要的。按照如下方法可以避免一些常见的错误：
通常往外部系统写入数据需要创建一个connection对象，并使用它来发送数据。为了达到这个目的，开发者往往不经意间会在Spark驱动程序中创建connection对象，然后在Spark worker中使用它。例如：
```python
def sendRecord(rdd):
    connection = createNewConnection()  # executed at the driver
    rdd.foreach(lambda record: connection.send(record))
    connection.close()

dstream.foreachRDD(sendRecord)
```
这种错误是由于要把connection对象从driver程序传到worker导致的。这类connection对象很少能在机器间传输。这种错误可能表现为序列化错误或初始化错误。正确的做法是在worker上创建connection对象。然而，这将导致另一个常见的错误——每条记录都创建一个新的connection对象：
```python
def sendRecord(record):
    connection = createNewConnection()
    connection.send(record)
    connection.close()

dstream.foreachRDD(lambda rdd: rdd.foreach(sendRecord))
```
通常创建一个connection对象需要消耗时间和资源，因此对于每个记录都要创建和销毁connection对象是耗费大量不必要的资源并且明显的降低系统吞吐量。更好的解决方法是使用rdd.foreachPartition——在一个RDD分区里仅创建一个connection对象来发送数据。
```python
def sendPartition(iter):
    connection = createNewConnection()
    for record in iter:
        connection.send(record)
    connection.close()

dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))
```
这个连接创建的消耗就可以平摊到多条记录了。
最后，还可以通过多个的RDDs/批处理来重用connection对象以进一步优化。可以维护一个静态连接池，在多个往外部系统传送数据的RDD之间可以重用，以此来减少系统消耗。
```python
def sendPartition(iter):
    # ConnectionPool is a static, lazily initialized pool of connections
    connection = ConnectionPool.getConnection()
    for record in iter:
        connection.send(record)
    # return to the pool for future reuse
    ConnectionPool.returnConnection(connection)

dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))
```
注意这些链接应该在需要的时候延迟创建，并且设置超时在长时间不用时销毁。这是将数据发送到外部系统最有效的方式。

###### 其他需要注意的点 ######
* DStreams会在执行输出操作时被延迟执行，就像只有RDD actions才会触发RDDs执行。具体说，就是DStream输出操作中的RDD actions会触发接收到的数据被处理。因此，如果你的程序里边没有输出操作，或者dstream.foreachRDD()中没有RDD action，程序将不会别执行。系统将在接收数据后直接丢弃。
* 默认情况下，输出操作只执行一次，且按照程序中定义的顺序。

#### <a NAME="t3-7">DataFrame和SQL操作</a>  ####
你可以在streaming中很容易的使用DataFrame和SQL。你只需要使用SparkContext创建一个SQLContext供StreamingContext使用即可。而且它可以在驱动程式失败的时候重启，这通过创建一个延迟实例化的单例SQLContext实现。下边例子将会展示。它是通过修改之前的例子，使用DataFrame和SQL来生成单词数量的。每一个RDD被转换成DataFrame，注册成临时表并通过sql查询。
```python
# Lazily instantiated global instance of SQLContext
def getSqlContextInstance(sparkContext):
    if ('sqlContextSingletonInstance' not in globals()):
        globals()['sqlContextSingletonInstance'] = SQLContext(sparkContext)
    return globals()['sqlContextSingletonInstance']

...

# DataFrame operations inside your streaming program

words = ... # DStream of strings

def process(time, rdd):
    print "========= %s =========" % str(time)
    try:
        # Get the singleton instance of SQLContext
        sqlContext = getSqlContextInstance(rdd.context)

        # Convert RDD[String] to RDD[Row] to DataFrame
        rowRdd = rdd.map(lambda w: Row(word=w))
        wordsDataFrame = sqlContext.createDataFrame(rowRdd)

        # Register as table
        wordsDataFrame.registerTempTable("words")

        # Do word count on table using SQL and print it
        wordCountsDataFrame = sqlContext.sql("select word, count(*) as total from words group by word")
        wordCountsDataFrame.show()
    except:
        pass

words.foreachRDD(process)
```
完整代码参考[源代码](https://github.com/apache/spark/blob/master/examples/src/main/python/streaming/sql_network_wordcount.py)

你同样可以使用sql查询定义在其他线程的streaming数据中的表（也就是说，异步运行StreamingContext）。只要保证StreamingContext村处理足够多的streaming数据以便查询运行即可。否则StreamingContext没有察觉到任何异步查询，将会在查询完成前删除旧数据。例如，你想查询最后一个批处理，但是查询会运行5分钟，那么可以使用streamingContext.remember(Minutes(5))。
DataFrame更多信息参考[DataFrames and SQL](http://spark.apache.org/docs/1.4.1/sql-programming-guide.html)

#### <a NAME="t3-8">MLlib操作</a> ####