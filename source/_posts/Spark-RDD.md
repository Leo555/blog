---
title: Spark RDD 基础
date: 2016-11-29 11:29:07
tags:
- Spark
- Scala
- RDD
categories: Big Data
---

# RDD 是什么？


<img src="/assets/img/spark-stack.png" alt="rdd">
[图片摘自[Spark 官网](http://spark.apache.org/)]

[RDD](http://spark.apache.org/docs/latest/programming-guide.html) 全称 **Resilient Distributed Datasets**，是 Spark 中的抽象数据结构类型，任何数据在Spark中都被表示为RDD。 Spark 建立在统一抽象的RDD之上，使得它可以以基本一致的方式应对不同的大数据处理场景，包括MapReduce，Streaming，SQL，Machine Learning 等。

<!-- more -->
简单的理解就是 RDD 就是一个数据结构，不过这个数据结构中的数据是分布式存储的，Spark 中封装了对 RDD 的各种操作，可以让用户显式地将数据存储到磁盘和内存中，并能控制数据的分区。


## RDD 特性

RDD 是 Spark 的核心，也是整个 Spark 的架构基础。它的特性可以总结如下：

- 它是不变的数据结构存储
- 它是支持跨集群的分布式数据结构
- 可以根据数据记录的key对结构进行分区
- 提供了粗粒度的操作，且这些操作都支持分区
- 它将数据存储在内存中，从而提供了低延迟性

# 创建 RDD

本文中的例子全部基于 [Spark-shell](http://spark.apache.org/downloads.html)，需要的请自行安装。

创建 RDD 主要有两种方式，一种是使用 SparkContext 的 **parallelize** 方法创建并行集合，还有一种是通过外部外部数据集的方法创建，比如本地文件系统，HDFS，HBase，Cassandra等。

## 并行集合

使用 parallelize 方法从普通数组中创建 RDD:

```shell
scala> val a = sc.parallelize(1 to 9, 3)
a: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:21
```

parallelize 方法接受两个参数，第一个是数据集合，第二个是切片的个数，表示将数据存放在几个分区中。

一旦创建完成，这个分布式数据集(a)就可以被并行操作。例如，我们可以调用 a.reduce((m, n) => m + n) 将这个数组中的元素相加。 更多的操作请见 [Spark RDD 操作](https://lz5z.com/rdd-operations)。


## 本地文件

文本文件 RDDs 可以使用 SparkContext 的 textFile 方法创建。 在这个方法里传入文件的 URI (机器上的本地路径或 hdfs://，s3n:// 等)，然后它会将文件读取成一个行集合。

读取文件 test.txt 来创建RDD，文件中的每一行就是RDD中的一个元素。

```shell
scala> val b = sc.textFile("test.txt")
b: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[2] at textFile at <console>:21
```

一旦创建完成，(b) 就能做数据集操作。例如，我们可以用下面的方式使用 map 和 reduce 操作将所有行的长度相加： b.map(s => s.length).reduce((m, n) => m + n)

```shell
scala> b.collect
res1: Array[String] = Array(Spark, RDD, Transformations, Actions)

scala> b.map(s => s.length).reduce((m, n) => m + n))
res2: Int = 30
```

### Spark 读文件注意事项

1. 如果使用本地文件系统路径，文件必须能在 worker 节点上用相同的路径访问到。要么复制文件到所有的 worker 节点，要么使用网络的方式共享文件系统。

2. 所有 Spark 的基于文件的方法，包括 textFile，能很好地支持文件目录，压缩过的文件和通配符。例如，你可以使用 textFile("/文件目录")，textFile("/文件\*.txt") 和 textFile("/文件目录/\*.gz")。

3. textFile 方法也可以选择第二个可选参数来控制切片(slices)的数目。默认情况下，Spark 为每一个文件块(HDFS 默认文件块大小是 64M)创建一个切片(slice)。但是你也可以通过一个更大的值来设置一个更高的切片数目。注意，你不能设置一个小于文件块数目的切片值。


### ScalaAPI 对其它数据格式的支持
1. SparkContext.wholeTextFiles 让你读取一个包含多个小文本文件的文件目录并且返回每一个(filename, content)对。与 textFile 的差异是：它记录的是每个文件中的每一行。

2. 对于 [SequenceFiles](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)，可以使用 SparkContext 的 sequenceFile[K, V] 方法创建，K 和 V 分别对应的是 key 和 values 的类型。像 [IntWritable](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/IntWritable.html) 与 [Text](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/Text.html) 一样，它们必须是 Hadoop 的 [Writable](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/Writable.html) 接口的子类。另外，对于几种通用的 Writables，Spark 允许你指定原生类型来替代。例如： sequenceFile[Int, String] 将会自动读取 IntWritables 和 Text。

3. 对于其他的 Hadoop InputFormats，你可以使用 SparkContext.hadoopRDD 方法，它可以指定任意的 JobConf，输入格式(InputFormat)，key 类型，values 类型。你可以跟设置 Hadoop job 一样的方法设置输入源。你还可以在新的 MapReduce 接口(org.apache.hadoop.mapreduce)基础上使用 SparkContext.newAPIHadoopRDD(译者注：老的接口是 SparkContext.newHadoopRDD)。

4. RDD.saveAsObjectFile 和 SparkContext.objectFile 支持保存一个RDD，保存格式是一个简单的 Java 
对象序列化格式。这是一种效率不高的专有格式，如 Avro，它提供了简单的方法来保存任何一个 RDD。