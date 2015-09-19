#目录

* 概述
* 一个简单的例子
* 基本概念
	* 相关依赖
	* 初始化StreamingContext
	* 离散流(DStream)
	* 输入流和接收器
	* DStream中的转换
	* DStream的输出操作
	* DataFrame和SQL操作
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

##概述
Spark Streaming是一个基于Spark核心API实现的易拓展、高效、可靠的实时流数据处理拓展组件。Spark的数据可以从多种数据源获取，如Kafka, Flume, Twitter, ZeroMQ, Kinesis或TCP sockets，并且可以通过使用map, reduce, join and window等高级函数进行复杂的算法处理，最终处理过的数据可以写入文件系统、数据库或者实时可视化报表。实际上，你还能在流处理的过程中使用Spark的机器学习和图处理算法。
![](http://spark.apache.org/docs/latest/img/streaming-arch.png)
从内部看，他的工作原理如下图。Spark Streaming接收输入数据流，并将这些数据分割成多个batch（这里直接用batch，感觉批次听着挺怪），然后再由Spark engine处理并生成最终的结果。
![](http://spark.apache.org/docs/latest/img/streaming-flow.png)
Spark Streaming提供了一个叫做离散流(DStream)的高级抽象，用来表示连续的流数据。DStreams既可以从Kafka、Flume、Kinesis这样源数据的输入流创建，也可以从高级操作中或者其他DStreams中产生。从内部看，DStreams用来表示一个RDD序列。
本指南可以告诉你如何使用DStreams来开发Spark Streaming程序。你可以使用Scala、Java或者Python编写程序，所有版本这里都会提供。
	注意：在Python中有一些API不同或者不可用，这些地方将会高亮标记。

##一个简单的例子
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

##基本概念
接下来，我们通过回顾之前那个简单的例子，来详细的说明Spark Streaming的基本概念。

####相关依赖
与Spark相似，Spark Streaming的依赖包可以从Maven仓库获取。你需要把下边的依赖加入你的Maven或SBT项目，以开发你自己的程序。
```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.10</artifactId>
    <version>1.4.1</version>
</dependency>
```
为了从Kafka、Flume和Kinesis这样未包含在核心API中的数据源获取数据，你还需要将相应的模块spark-streaming-xyz_2.10加入到依赖中。例如，下边这些组件：
![](https://github.com/NotBadPad/translation/blob/master/bigdate/img/spark-1.png)
若要获取最新的列表，请参考maven仓库中完整的支持的列表。

####初始化StreamingContext