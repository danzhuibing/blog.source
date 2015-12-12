title: Spark入门&大数据处理技术比较 
date: 2015-10-10 07:07:36
tags:
- Spark
- Python
- 大数据
categories: 大数据基础

---

## RDD的概念
RDD是Spark的核心。关于它的定义，简单来说，就是能够保存在内存里的分布式数据集。官方定义如下：

> The main abstraction Spark provides is a **resilient distributed dataset (RDD)**, which is a collection of elements partitioned across the nodes of the cluster that can be operated on in parallel.    
出处：*Spark Programming Guide*


对于Hadoop而言，分布式数据集是存储在文件系统HDFS上的，这样的坏处：数据处理往往需要多个MapReduce环节，下一个环节很可能依赖于上一个环节的输出；由于没有分布式内存机制，Hadoop必须把上一个环节MapReduce的输出结果写到硬盘HDFS上，下一个环节再从HDFS上读取。因此磁盘IO将会花费大量的时间。这种模式对于迭代式计算，如机器学习，非常不便。

Spark创造的RDD机制可以很好地解决这个问题。一般而言，Spark的流程如下：首先，从HDFS读取数据创建RDD；然后对RDD进行各种数据转换与处理的操作（可以简单理解为若干个Hadoop的MapReduce操作），直到得到满意的结果；最后，将结果写到HDFS，或者输出至屏幕。

如果把Spark的这个过程和Hadoop作对比，最大的区别就是用RDD替代了HDFS作为不同数据处理环节的媒介；当然，还有一个重要的区别，Hadoop只提供了Map和Reduce两种操作，Spark提供了更加灵活的操作方法，后面会有更详细的阐述。

## 入门例子：wordcount
``` python
from pyspark import SparkContext
sc = SparkContext("local", "wordcount")
textFile = sc.textFile("YOUR_SPARK_HOME/README.md")
wordCounts = textFile.flatMap(lambda line: line.split()).map(lambda word: (word, 1)).reduceByKey(lambda a, b: a+b)
print wordCounts.collect()
```

代码剖析如下：
1. 导入Python使用的Spark包
2. 初始化Spark环境，"local"表示程序用单机模拟Spark，"wordcount"是Spark程序的名字，这个名字会显示在Spark的集群管理页面
3. **textFile()**用于从文件系统读取数据，生成RDD
4. **flatMap()**将RDD的每一行按照参数定义的函数转化为多行，*lambda*定义了Python的匿名函数，函数输入是line，输出是line.split()；**map()**将RDD的每一行按照参数定义的函数转化为一行，(word, 1)表示一个键值对，word为key，1为value；**reduceByKey()**将RDD按照key分组后，按照参数定义的函数进行合并，此处函数的作用是是分组求和
5. **collect()**，将RDD的数据，从集群的各个节点收集回提交任务的机器，放在单机内存里

## RDD Operations和持久化

关于RDD的Operations可以分为两类，一类是**transformation**，作用是从一个RDD转变成新的RDD，如例子中的flatMap()，map()，reduceByKey()；一类是**action**，对RDD做计算，把结果返回给提交任务的机器，如例子中的collect()。值得注意的是，所有的transformation都是*lazy*机制，即transformation不会立刻执行，只是被程序记住要执行这个操作，直到碰到action时，才按照记录的操作顺序触发数据处理流程。

每个action触发计算时，相关的transformation会重新计算；可以用**persist()**在内存中保存中间RDD，这样下次action触发时，就可以从persist后的RDD开始，节省计算时间。如果要释放空间，可以调用**unpersist()**。

## 大数据数据处理技术比较
个人认为，SQL是数据处理的典范。评价一个数据处理工具最好的方法就是拿工具提供的功能和SQL进行对比。下面是我个人对Hive、Spark和Hadoop三个技术的比较。

| **Hive** | **Spark** | **Hadoop** |
| ---- | ----- | ------ |
| where | **filter()** | Map |
| join | **join()** | MapReduce |
| group by | **groupByKey()** | MapReduce |
| distribute by, sort by, cluster by | **repartitionAndSortWithinPartitions(partitioner)** | Shuffle
| distinct | **distinct()** | MapReduce |
| udf | **map()** | Map |
| udaf | **reduce()** | Map |
| udtf | **flatMap()** | Map |
| 动态分区 | 原生API不支持, 版本号1.5.2 | MapReduce |

Hadoop是Hive实现的基础，所有Hive撰写的SQL代码最终都会转化为底层的MapReduce作业。所以Hadoop是支持所有SQL需要的功能。问题在于，Hadoop就只提供了Map和Reduce的编程接口，这是非常底层的一个接口。本质上，Map和Reduce是一样的，就是一个自定义函数，告诉Hadoop怎么处理用户的数据；如果这个自定义函数不需要shuffle，那么就是Map；如果这个自定义函数之前需要shuffle，那么就是Reduce。

然而，不管用户的数据长什么样子，数据处理的模式就那几种，基本都涵盖在SQL的关键字表达里。Hive简单来讲，就是在底层先用MapReduce实现了可复用的SQL各种关键字对应的数据处理模式，在顶层提供给用户一个类SQL的编程接口，在中间通过一个解析器把顶层用户撰写的SQL语句转变为对底层可复用的MapReduce代码的调用。

Spark的批处理模式与Hadoop在理念上是一致的，也是MapReduce模型，也提供了shuffle机制；不同的是，用RDD替换了HDFS作为数据处理环节的媒介。另一方面，Spark与Hive类似，提供了比Hadoop MapReduce丰富得多的数据处理模式，即RDD的Operations，基本上与Hive SQL提供的功能相同。

## Spark和Hadoop的shuffle比较

此处需专门拉出一文讨论，可先参考我同事做的这个笔记[Spark Shuffle详解](http://xialeizhou.com/2015/12/08/Spark-Shuffle%E8%AF%A6%E8%A7%A3/)。总的来讲，我感觉是两个大区别：
- Map端：Hadoop现在是一个mapper输出一个文件；Spark是一个mapper输出R个文件，R为reducer的个数。
- Reduce端：Hadoop从Map端抓来属于自己分组的数据后，会做一次强制的merge sort；Spark从Map端抓来属于自己分组的数据后，不强制做sort。Hadoop强制做了merge sort的好处是，相同key的数据会被reducer一次性连续访问完，因此做reduce操作时只需record by record地聚合，而不用消耗内存；坏处是，并不是所有的reduce操作都需要按照key进行排序。Spark的优劣势反之，不排序的后果是在做reduce时，必须在内存开一个dict来存聚合的结果，对内存消耗大，可能撑爆内存。


## 小结
Spark在理念上和Hadoop如出一辙，都是基于MapReduce的思想，基于shuffle的。一方面，Spark实现了绝大部分数据处理会用到的MapReduce模式，提供了更丰富的语义接口供用户调用，使得用户在完成相同的数据处理流程时撰写的代码量大幅度减少。另一方面，Spark独创的RDD使中间结果能够放在内存中而不用写到HDFS，这样减少了IO时间的消耗，数据处理延时低，除了批处理应用场景外，还适合迭代式计算（Spark MLlib, Spark GraphX）、交互式数据处理（Spark SQL）、实时数据处理（Spark Streaming）。

