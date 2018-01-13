---
title: RDD 编程指南
date: 2018-01-12 22:23:30
tags:
categories: spark
---

## 概观
在高层次上，每个Spark应用程序都包含一个驱动程序，该程序运行用户的main功能，并在集群上执行各种并行操作。Spark提供的主要抽象是一个弹性分布式数据集（RDD），它是在集群节点间进行分区的元素集合，可以并行操作。RDD是通过从Hadoop文件系统（或任何其他Hadoop支持的文件系统）中的文件或驱动程序中现有的Scala集合开始创建的，并对其进行转换。用户还可以要求火花持续存储器中的RDD，允许其有效地跨越并行操作被重复使用。最后，RDD自动从节点故障中恢复。

Spark中的第二个抽象是可用于并行操作的共享变量。默认情况下，Spark在不同节点上并行执行一组任务时，会将该函数中使用的每个变量的副本传送给每个任务。有时候，变量需要在任务之间，或任务与驱动程序之间共享。Spark支持两种类型的共享变量：广播变量，可用于在所有节点上缓存内存中的值，以及累加器，这些变量只是“添加”到的变量，如计数器和总和。

本指南显示了Spark支持的各种语言中的每个功能。如果您启动Spark的交互式shell（无论bin/spark-shell是Scala shell还是 bin/pysparkPython的），最容易跟随。

## 链接到Spark
Spark 2.2.1是构建和分发的，默认使用Scala 2.11。（Spark也可以与Scala的其他版本一起工作。）要在Scala中编写应用程序，您需要使用兼容的Scala版本（例如2.11.X）。

要编写Spark应用程序，您需要在Spark上添加Maven依赖项。Spark可以通过Maven Central获得：

```sbt
groupId = org.apache.spark
artifactId = spark-core_2.11
version = 2.2.1
```
另外，如果您希望访问HDFS集群，则需要hadoop-client为您的HDFS版本添加依赖项 。

```
groupId = org.apache.hadoop
artifactId = hadoop-client
version = <your-hdfs-version>
```
最后，您需要将一些Spark类导入到您的程序中。添加以下行：

```scala
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
```

（在Spark 1.3.0之前，您需要明确`import org.apache.spark.SparkContext._`地启用基本的隐式转换。）

## 初始化Spark
Spark程序必须做的第一件事就是创建一个SparkContext对象，它告诉Spark如何访问一个集群。要创建一个SparkContext你首先需要建立一个包含你的应用程序信息的SparkConf对象。

每个JVM只能有一个SparkContext处于活动状态。stop()在创建一个新的SparkContext之前，您必须使用活动的SparkContext。

```scala
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```
该appName参数是您的应用程序在集群UI上显示的名称。 master是Spark，Mesos或YARN群集URL，或者是以本地模式运行的特殊“本地”字符串。实际上，在群集上运行时，您不希望master在程序中进行硬编码，而是启动应用程序spark-submit并在其中接收应用程序。但是，对于本地测试和单元测试，您可以通过“本地”来运行进程中的Spark。

在Spark shell中，已经为您创建了一个特殊的解释器感知的SparkContext，它被称为变量sc。制作自己的SparkContext将不起作用。您可以使用--master参数来设置上下文所连接的主机，并且可以通过将逗号分隔列表传递给参数来将JAR添加到类路径中--jars。您还可以通过向参数提供逗号分隔的Maven坐标列表来将依赖关系（例如Spark包）添加到shell会话中--packages。可能存在依赖关系的任何附加存储库（例如Sonatype）都可以传递给--repositories参数。例如，要bin/spark-shell在四个内核上运行，请使用：

```
$ ./bin/spark-shell --master local[4]
```
或者，也要添加code.jar到其类路径中，请使用：

```
$ ./bin/spark-shell --master local[4] --jars code.jar
```
要使用Maven坐标包含依赖项：

```scala
$ ./bin/spark-shell --master local[4] --packages "org.example:example:0.1"
```
有关选项的完整列表，请运行spark-shell --help。在幕后， spark-shell调用更一般的spark-submit脚本。

## 弹性分布式数据集（RDD）
Spark围绕弹性分布式数据集（RDD）的概念展开，RDD是可以并行操作的容错元素集合。有两种方法可以创建RDD：并行化 驱动程序中的现有集合，或在外部存储系统（如共享文件系统，HDFS，HBase或提供Hadoop InputFormat的任何数据源）中引用数据集。

### 并行集合
并行化集合是通过调用驱动程序（Scala ）中现有集合上SparkContext的parallelize方法来创建的Seq。集合的元素被复制以形成可以并行操作的分布式数据集。例如，下面是如何创建一个保存数字1到5的并行化集合：

```scala
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
```
一旦创建，分布式数据集（distData）可以并行操作。例如，我们可能调用distData.reduce((a, b) => a + b)将数组的元素相加。我们稍后介绍分布式数据集上的操作。

并行收集的一个重要参数是要将数据集剪切成的分区数量。Spark将为群集的每个分区运行一个任务。通常情况下，您需要为群集中的每个CPU分配2-4个分区。通常情况下，Spark会尝试根据您的群集自动设置分区数量。但是，也可以通过将其作为第二个参数传递给parallelize（eg sc.parallelize(data, 10)）来进行手动设置。注意：代码中的一些地方使用术语切片（分区的同义词）来维持向后兼容性。

### 外部数据集
Spark可以从Hadoop支持的任何存储源（包括本地文件系统，HDFS，Cassandra，HBase，Amazon S3等）创建分布式数据集.Spark支持文本文件，SequenceFile和任何其他Hadoop InputFormat。

文本文件RDDS可以使用创建SparkContext的textFile方法。此方法需要一个URI的文件（本地路径的机器上，或一个hdfs://，s3n://等URI），并读取其作为行的集合。这是一个示例调用：

```scala
scala> val distFile = sc.textFile("data.txt")
distFile: org.apache.spark.rdd.RDD[String] = data.txt MapPartitionsRDD[10] at textFile at <console>:26
```
一旦创建，distFile可以通过数据集操作进行操作。例如，我们可以使用map和reduce操作加起来所有行的大小如下：`distFile.map(s => s.length).reduce((a, b) => a + b)`。

使用Spark读取文件的一些注意事项：

如果在本地文件系统上使用路径，则该文件也必须可以在工作节点上的相同路径上访问。将文件复制到所有工作人员或使用网络安装的共享文件系统。

Spark的所有基于文件的输入方法，包括textFile支持在目录，压缩文件和通配符上运行。例如，你可以使用`textFile("/my/directory")，textFile("/my/directory/*.txt")和textFile("/my/directory/*.gz")`。

该textFile方法还采用可选的第二个参数来控制文件的分区数量。默认情况下，Spark为文件的每个块创建一个分区（HDFS中的块默认为128MB），但是您也可以通过传递更大的值来请求更多的分区。请注意，您不能有比块更少的分区。

除了文本文件外，Spark的Scala API还支持其他几种数据格式：

- SparkContext.wholeTextFiles让你阅读一个包含多个小文本文件的目录，并将它们作为（文件名，内容）对返回。这与textFile每个文件每行返回一条记录相反。分区由数据局部性决定，在某些情况下可能导致分区太少。对于这些情况，wholeTextFiles提供一个可选的第二个参数来控制分区的最小数量。

- 对于SequenceFiles，使用SparkContext的sequenceFile[K, V]方法，其中K和V是文件中的键和值的类型。这些应该是Hadoop的Writable接口的子类，如IntWritable和Text。另外，Spark允许您为几个常见Writable指定本机类型; 例如，sequenceFile[Int, String]将自动读取IntWritables和文本。

- 对于其他Hadoop InputFormats，您可以使用该SparkContext.hadoopRDD方法，该方法采用任意的JobConf输入格式类，关键类和值类。将它们设置为您使用输入源进行Hadoop作业的方式相同。您也可以使用SparkContext.newAPIHadoopRDD基于“新”MapReduce API（org.apache.hadoop.mapreduce）的InputFormats 。

- RDD.saveAsObjectFile并SparkContext.objectFile支持以包含序列化Java对象的简单格式保存RDD。虽然这不像Avro这样的专业格式，但它提供了一种简单的方法来保存任何RDD。

### RDD操作
RDDS支持两种类型的操作：转变，从现有的创建一个新的数据集和行动，其上运行的数据集的计算后的值返回驱动程序。例如，map是一个通过函数传递每个数据集元素的变换，并返回表示结果的新RDD。另一方面，reduce是一个动作，使用某个函数来聚合RDD的所有元素，并将最终结果返回给驱动程序（尽管也有一个并行reduceByKey返回分布式数据集）。

Spark中的所有转换都是懒惰的，因为它们不会马上计算结果。相反，他们只记得应用于某些基础数据集（例如文件）的转换。只有在动作需要将结果返回给驱动程序时才会计算转换。这种设计使Spark能够更高效地运行。例如，我们可以认识到，通过创建的数据集map将被用于a中，reduce并且只返回reduce给驱动程序的结果，而不是更大的映射数据集。

默认情况下，每次对其执行操作时，每个已转换的RDD都可能重新计算。但是，您也可以使用（或）方法将RDD 保留在内存中，在这种情况下，Spark将保留群集中的元素，以便在下次查询时快速访问。还支持在磁盘上持久化RDD，或在多个节点上复制RDD。persistcache

#### 基本
为了说明RDD基础知识，请考虑下面的简单程序：

```scala
val lines = sc.textFile("data.txt")
val lineLengths = lines.map(s => s.length)
val totalLength = lineLengths.reduce((a, b) => a + b)
```
第一行定义了来自外部文件的基本RDD。这个数据集不会被加载到内存中或者作用于其他地方：lines仅仅是一个指向文件的指针。第二行定义lineLengths为map转换的结果。再次，lineLengths 是不是马上计算，由于懒惰。最后，我们跑reduce，这是一个行动。在这一点上，Spark将计算分解为在不同机器上运行的任务，每台机器既运行其地图部分又运行局部缩减，只返回驱动程序的答案。

如果我们还想以后再使用lineLengths，可以添加：

lineLengths.persist()
之前reduce，这将导致lineLengths在第一次计算后保存在内存中。

#### 将函数传递给Spark
Spark的API在很大程度上依赖于将驱动程序中的函数传递到集群上运行。有两种建议的方法来做到这一点：

匿名函数的语法，可用于短小的代码。
全局单例对象中的静态方法 例如，您可以定义object MyFunctions并传递MyFunctions.func1，如下所示：

```scala
object MyFunctions {
  def func1(s: String): String = { ... }
}

myRdd.map(MyFunctions.func1)
```

请注意，虽然也可以在类实例中传递对方法的引用（与单例对象相反），但这需要将包含该类的对象与方法一起发送。例如，考虑：

```scala
class MyClass {
  def func1(s: String): String = { ... }
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
}
```
在这里，如果我们创建一个新的MyClass实例并对其进行调用doStuff，那么其中的map内部引用了该实例的 func1方法，因此需要将整个对象发送到集群。这与写作相似。MyClassrdd.map(x => this.func1(x))

以类似的方式，访问外部对象的字段将引用整个对象：
```scala
class MyClass {
  val field = "Hello"
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
}
```
相当于写作rdd.map(x => this.field + x)，其中引用所有的this。为了避免这个问题，最简单的方法是复制field到本地变量而不是从外部访问：

```scala
def doStuff(rdd: RDD[String]): RDD[String] = {
  val field_ = this.field
  rdd.map(x => field_ + x)
}
```
## 了解闭包
Spark的难点之一是在集群中执行代码时理解变量和方法的范围和生命周期。修改范围之外的变量的RDD操作可能是混淆的常见来源。在下面的例子中，我们将看看foreach()用于递增计数器的代码，但是其他操作也会出现类似的问题。

### 例
考虑下面的天真的RDD元素总和，根据执行是否发生在同一个JVM中，这可能会有不同的表现。一个常见的例子是在local模式（--master = local[n]）中运行Spark 而不是将Spark应用程序部署到集群（例如，通过spark-submit to YARN）：

```scala
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
```

### 本地或集群模式
上面的代码的行为是未定义的，并可能无法正常工作。为了执行作业，Spark将RDD操作的处理分解为任务，每个任务由执行者执行。在执行之前，Spark计算任务的关闭。闭包是执行者在RDD上执行其计算的那些变量和方法（在这种情况下foreach()）。这个封闭序列化并发送给每个执行者。

发送给每个执行程序的闭包内的变量现在是副本，因此，当函数内引用计数器时foreach，它不再是驱动程序节点上的计数器。驱动程序节点的内存中还有一个计数器，但执行程序不再可见！执行者只能看到序列化闭包的副本。因此，计数器的最终值仍然是零，因为计数器上的所有操作都引用了序列化闭包内的值。

在本地模式下，在某些情况下，该foreach函数将在与驱动程序相同的JVM中实际执行，并将引用相同的原始计数器，并可能实际更新它。

为了确保在这些场景中明确定义的行为，应该使用一个Accumulator。Spark中的累加器专门用于提供一种在集群中的工作节点之间执行拆分时安全地更新变量的机制。本指南的“累加器”部分更详细地讨论了这些内容。

一般来说，闭包 - 像循环或本地定义的方法这样的构造不应该被用来改变一些全局状态。Spark并没有定义或保证对从封闭外引用的对象的突变行为。这样做的一些代码可能在本地模式下工作，但这是偶然的，这样的代码不会按预期在分布式模式下运行。如果需要全局聚合，请使用累加器。

### 打印RDD的元素
另一个常见的习惯是试图用rdd.foreach(println)or 打印RDD的元素rdd.map(println)。在一台机器上，这将产生预期的输出并打印所有RDD的元素。然而，在cluster模式下，stdout执行者所调用的输出现在是写给执行者的，stdout而不是驱动器上的，所以stdout驱动程序不会显示这些！要打印驱动程序中的所有元素，可以使用该collect()方法首先将RDD带到驱动程序节点：rdd.collect().foreach(println)。这可能会导致驱动程序内存不足，因为collect()将整个RDD提取到一台机器; 如果您只需要打印RDD的一些元素，则更安全的方法是使用take()：rdd.take(100).foreach(println)。

## 使用键值对
虽然大多数Spark操作在包含任何类型对象的RDD上工作，但是一些特殊操作仅在键 - 值对的RDD上可用。最常见的是分布式的“随机”操作，如按键分组或聚合元素。

在Scala中，这些操作可以在包含Tuple2对象的RDD上自动使用 （语言中的内置元组，通过简单的书写创建(a, b)）。PairRDDFunctions类中提供了键值对操作， 该类会自动包装元组的RDD。

例如，以下代码使用reduceByKey键值对上的操作来计算文本中每行文本的出现次数：

```scala
val lines = sc.textFile("data.txt")
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
```
counts.sortByKey()例如，我们也可以按字母顺序对这些对进行排序，最后 counts.collect()把它们作为一个对象数组返回给驱动程序。

注意：在使用自定义对象作为键值对操作中的键时，必须确保自定义equals()方法附带匹配hashCode()方法。有关完整的详细信息，请参阅Object.hashCode（）文档中概述的合同。

## Transform
下表列出了Spark支持的一些常见转换。有关详细信息，请参阅RDD API文档（Scala， Java， Python， R）和RDD函数doc（Scala， Java）。

- map（func）	通过函数func传递源的每个元素来形成一个新的分布式数据集。
- filter（func）	通过选择func返回true 的源的元素返回一个新的数据集。
- flatMap（func）	类似于map，但是每个输入项可以映射到0个或更多个输出项（所以func应该返回一个Seq而不是单个项）。
- mapPartitions（func）	与map类似，但是在RDD的每个分区（块）上分别运行，所以当在T型RDD上运行时，func必须是Iterator <T> => Iterator <U>类型。
- mapPartitionsWithIndex（func）	类似于mapPartitions，而且还提供FUNC与表示所述分区的索引的整数值，所以FUNC必须是类型（中间体，迭代器<T>）=>的迭代器上类型T的RDD运行时<U>
- sample（与更换，分数，种子）	使用给定的随机数发生器种子对数据的一小部分进行采样，有或没有替换。
- union（otherDataset）	返回包含源数据集中的元素和参数的联合的新数据集。
- intersection（otherDataset）	返回一个新的RDD，其中包含源数据集中的元素和参数的交集。
- distinct（[ numTasks ]））	返回包含源数据集的不同元素的新数据集。
- groupByKey（[ numTasks ]）	当调用（K，V）对的数据集时，返回（K，Iterable <V>）对的数据集。
注意：如果您正在对每个密钥执行汇总（例如总计或平均），则使用reduceByKey或aggregateByKey将产生更好的性能。
注：默认情况下，输出中的并行级别取决于父RDD的分区数量。您可以传递可选numTasks参数来设置不同数量的任务。
- reduceByKey（func，[ numTasks ]）	当调用（K，V）对的数据集时，返回（K，V）对的数据集，其中每个键的值使用给定的reduce函数func进行聚合，函数func必须是（V，V）=> V.像in一样groupByKey，reduce任务的数量可以通过可选的第二个参数来配置。
- aggregateByKey（zeroValue）（seqOp，combOp，[ numTasks ]）	当调用（K，V）对的数据集时，返回（K，U）对的数据集，其中使用给定的组合函数和中性的“零”值来汇总每个键的值。允许与输入值类型不同的聚合值类型，同时避免不必要的分配。像in一样groupByKey，reduce任务的数量可以通过可选的第二个参数来配置。
- sortByKey（[ ascending ]，[ numTasks ]）	当调用K实现Ordered的（K，V）对的数据集时，按照布尔ascending参数中的指定，按照升序或降序返回按键排序的（K，V）对的数据集。
- join（otherDataset，[ numTasks ]）	当（K，V）和（K，W）类型的数据集被调用时，返回（K，（V，W））对的每个键的所有元素对的数据集。外连接通过支持leftOuterJoin，rightOuterJoin和fullOuterJoin。
- cogroup（otherDataset，[ numTasks ]）	当（K，V）和（K，W）类型的数据集被调用时，返回（K，（Iterable <V>，Iterable <W>））元组的数据集。这个操作也被称为groupWith。
- cartesian（otherDataset）	当调用类型T和U的数据集时，返回（T，U）对（所有元素对）的数据集。
- pipe（命令，[envVars]）	通过shell命令管理RDD的每个分区，例如Perl或bash脚本。RDD元素被写入进程的stdin，输出到stdout的行被作为字符串的RDD返回。
- coalesce（numPartitions）	减少RDD中的分区数量为numPartitions。用于在过滤大型数据集后更高效地运行操作。
- repartition（numPartitions）	随机调整RDD中的数据以创建更多或更少的分区并在其间进行平衡。这总是通过网络混洗所有数据。
- repartitionAndSortWithinPartitions（partitioner）	根据给定的分区器对RDD进行重新分区，并在每个结果分区中按键分类记录。这比repartition在每个分区中调用然后排序更高效，因为它可以将排序推送到洗牌机器中。

## Actions
下表列出了Spark支持的一些常用操作。请参阅RDD API文档（Scala， Java， Python， R）

并配对RDD函数doc（Scala， Java）以获取详细信息。

行动	含义
减少（func）	使用函数func（它接受两个参数并返回一个）来聚合数据集的元素。该函数应该是可交换和关联的，以便它可以被正确地并行计算。
collect（）	在驱动程序中将数据集的所有元素作为数组返回。在过滤器或其他操作返回足够小的数据子集之后，这通常很有用。
count（）	返回数据集中元素的数量。
第一（）	返回数据集的第一个元素（类似于take（1））。
拿（n）	用数据集的前n个元素返回一个数组。
takeSample（withReplacement，num，[ seed ]）	返回一个阵列的随机样本NUM数据集的元素，具有或不具有取代，任选地预先指定的随机数发生器的种子。
takeOrdered（n，[订购]）	使用自然顺序或自定义比较器返回RDD 的前n个元素。
saveAsTextFile（路径）	将数据集的元素作为文本文件（或文本文件集）写入本地文件系统，HDFS或任何其他Hadoop支持的文件系统的给定目录中。Spark将在每个元素上调用toString将其转换为文件中的一行文本。
saveAsSequenceFile（路径）
（Java和Scala）	将数据集的元素作为Hadoop SequenceFile写入本地文件系统，HDFS或任何其他Hadoop支持的文件系统的给定路径中。这在实现Hadoop的Writable接口的键值对的RDD上是可用的。在Scala中，它也可用于可隐式转换为Writable的类型（Spark包含Int，Double，String等基本类型的转换）。
saveAsObjectFile（路径）
（Java和Scala）	使用Java序列化以简单的格式写入数据集的元素，然后可以使用Java序列化加载 SparkContext.objectFile()。
countByKey（）	仅适用于类型（K，V）的RDD。返回（K，Int）对的hashmap和每个键的计数。
foreach（func）	在数据集的每个元素上运行函数func。这通常用于副作用，如更新累加器或与外部存储系统交互。
注意：修改除“累加器”以外的变量foreach()可能会导致未定义的行为。请参阅了解更多细节。
Spark RDD API还公开了一些动作的异步版本，比如foreachAsyncfor foreach，它立即返回FutureAction给调用者，而不是完成动作时的阻塞。这可以用来管理或等待操作的异步执行。

## shuffle 操作
Spark中的某些操作会触发一个称为shuffle的事件。洗牌是Spark重新分配数据的机制，以便在不同分区之间进行分组。这通常涉及在执行者和机器之间复制数据，使得洗牌成为复杂而昂贵的操作。

### 背景
为了理解在洗牌过程中会发生什么，我们可以考虑reduceByKey操作的例子 。该reduceByKey操作将生成一个新的RDD，其中单个键的所有值都组合为一个元组 - 键和对与该键相关的所有值执行reduce函数的结果。面临的挑战是，并不是所有的单个密钥的值都必须位于同一个分区，甚至是同一个机器上，但是它们必须位于同一地点才能计算出结果。

在Spark中，数据通常不是跨分区分布，而是在特定操作的必要位置。在计算过程中，单个任务将在单个分区上运行 - 因此，要组织所有数据reduceByKey以执行单个reduce任务，Spark需要执行全部操作。它必须从所有分区中读取所有键的值，然后将各个分区上的值汇总在一起，以计算每个键的最终结果 - 这就是所谓的混洗。

虽然新洗牌数据的每个分区中的元素集合是确定性的，分区本身的排序也是确定性的，但这些元素的排序并不是这样。如果一个人在随机播放之后需要可预测的有序数据，那么可以使用：

mapPartitions 使用例如对每个分区进行排序， .sorted
repartitionAndSortWithinPartitions 在对分区进行有效分类的同时进行分区
sortBy 做一个全球有序的RDD
这可能会导致一个洗牌的操作包括重新分区一样操作 repartition和coalesce，ByKey”操作，比如（除计数）groupByKey，并reduceByKey和 参加操作，如cogroup和join。

### 性能影响
所述随机播放是昂贵的操作，因为它涉及的磁盘I / O，数据序列，和网络I / O。为了组织数据，Spark生成一组任务 - 映射任务来组织数据，以及一组reduce任务来聚合它。这个术语来自MapReduce，并不直接涉及到Spark map和reduce操作。

在内部，来自个别地图任务的结果被保存在内存中，直到它们不适合为止。然后，这些将根据目标分区进行排序并写入单个文件。在减少方面，任务读取相关的排序块。

某些随机操作会消耗大量的堆内存，因为它们使用内存中的数据结构来在传输之前或之后组织记录。具体而言， reduceByKey并aggregateByKey创建在地图上侧这样的结构，和'ByKey操作产生这些上减少侧。当数据不适合存储在内存中时，Spark会将这些表泄露到磁盘，导致额外的磁盘I / O开销和增加的垃圾回收。

Shuffle也会在磁盘上生成大量的中间文件。从Spark 1.3开始，这些文件将被保留，直到相应的RDD不再使用并被垃圾回收。这样做是为了在重新计算谱系时不需要重新创建洗牌文件。如果应用程序保留对这些RDD的引用，或者GC不经常引入，垃圾收集可能会在很长一段时间后才会发生。这意味着长时间运行的Spark作业可能会消耗大量的磁盘空间。临时存储目录spark.local.dir在配置Spark上下文时由配置参数指定 。

随机行为可以通过调整各种配置参数来调整。请参阅“ Spark配置指南 ”中的“Shuffle Behavior”部分。

### RDD持久性
Spark中最重要的功能之一就是在内存中持续（或缓存）一个数据集。当持久化RDD时，每个节点存储它在内存中计算的所有分区，并在该数据集的其他操作（或从中派生的数据集）中重用它们。这使未来的行动更快（通常超过10倍）。缓存是迭代算法和快速交互式使用的关键工具。

你可以用一个persist()或者多个cache()方法来标记一个RDD 。第一次在动作中计算时，它将被保存在节点的内存中。Spark的缓存是容错的 - 如果RDD的任何分区丢失，它将自动使用最初创建它的转换重新计算。

另外，每个持久RDD可以使用不同的存储级别进行存储，例如，允许您将数据集保存在磁盘上，将其保存在内存中，但是作为序列化的Java对象（以节省空间）将其复制到节点上。这些级别通过传递一个 StorageLevel对象（Scala， Java， Python）来设置persist()。该cache()方法是使用默认存储级别的简写，它是StorageLevel.MEMORY_ONLY（将反序列化对象存储在内存中）。全套存储级别是：

存储级别	含义
MEMORY_ONLY	将RDD作为反序列化的Java对象存储在JVM中。如果RDD不适合内存，某些分区将不会被缓存，并且每次需要时都会重新进行计算。这是默认级别。
MEMORY_AND_DISK	将RDD作为反序列化的Java对象存储在JVM中。如果RDD不适合内存，请存储不适合磁盘的分区，并在需要时从中读取。
MEMORY_ONLY_SER
（Java和Scala）	将RDD存储为序列化的 Java对象（每个分区一个字节的数组）。这通常比反序列化的对象更节省空间，特别是在使用 快速序列化器的情况下，但需要消耗更多的CPU资源。
MEMORY_AND_DISK_SER
（Java和Scala）	与MEMORY_ONLY_SER类似，但是将不适合内存的分区溢出到磁盘，而不是每次需要时重新计算它们。
DISK_ONLY	将RDD分区仅存储在磁盘上。
MEMORY_ONLY_2，MEMORY_AND_DISK_2等	与上面的级别相同，但复制两个群集节点上的每个分区。
OFF_HEAP（实验）	与MEMORY_ONLY_SER类似，但将数据存储在 堆内存中。这需要启用堆堆内存。
注意： 在Python中，存储对象将始终与Pickle库序列化，所以选择序列化级别无关紧要。Python中的可用存储级别包括MEMORY_ONLY，MEMORY_ONLY_2， MEMORY_AND_DISK，MEMORY_AND_DISK_2，DISK_ONLY，和DISK_ONLY_2。

reduceByKey即使没有用户调用，Spark也会自动保留一些中间数据（例如）persist。这样做是为了避免在洗牌过程中节点失败时重新输入整个输入。我们仍然建议用户调用persist所产生的RDD，如果他们打算重用它。

### 选择哪个存储级别？
Spark的存储级别旨在提供内存使用和CPU效率之间的不同折衷。我们建议通过以下过程来选择一个：

如果你的RDD适合默认的存储级别（MEMORY_ONLY），那就留下来吧。这是CPU效率最高的选项，允许RDD上的操作尽可能快地运行。

如果没有，尝试使用MEMORY_ONLY_SER和选择一个快速的序列化库，使对象更加节省空间，但仍然相当快地访问。（Java和Scala）

除非计算您的数据集的函数很昂贵，否则他们会过滤大量数据，否则不要泄露到磁盘。否则，重新计算分区可能与从磁盘读取分区一样快。

如果要快速恢复故障，请使用复制的存储级别（例如，如果使用Spark来为Web应用程序提供请求）。所有的存储级别通过重新计算丢失的数据来提供完整的容错能力，但是复制的容量可以让您继续在RDD上运行任务，而无需等待重新计算丢失的分区。

### 删除数据
Spark会自动监视每个节点上的高速缓存使用情况，并以最近最少使用（LRU）方式删除旧的数据分区。如果您想要手动删除RDD，而不是等待其从缓存中删除，请使用该RDD.unpersist()方法。

## 共享变量
通常，当传递给Spark操作（如mapor reduce）的函数在远程集群节点上执行时，它将在函数中使用的所有变量的单独副本上工作。这些变量被复制到每台机器上，远程机器上的变量没有更新传播到驱动程序。支持通用的，可读写的共享变量将是低效的。但是，Spark 为两种常见使用模式提供了两种有限类型的共享变量：广播变量和累加器。

### 广播变量
广播变量允许程序员在每台机器上保存一个只读变量，而不是用任务发送一个只读变量的副本。例如，可以使用它们以有效的方式为每个节点提供大型输入数据集的副本。Spark还尝试使用高效的广播算法来分发广播变量，以降低通信成本。

Spark动作是通过一系列阶段执行的，由分散的“随机”操作分开。Spark会自动播放每个阶段中任务所需的通用数据。以这种方式广播的数据以序列化形式缓存，并在运行每个任务之前反序列化。这意味着只有跨多个阶段的任务需要相同的数据或以反序列化的形式缓存数据时，显式创建广播变量才是有用的。

广播变量是v通过调用从一个变量创建的SparkContext.broadcast(v)。广播变量是一个包装器v，它的值可以通过调用value 方法来访问。下面的代码显示了这一点：

```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```
在创建广播变量之后，应该使用它来代替v群集上运行的任何函数中的值，以便v不会多次将其发送到节点。另外，v为了确保所有节点获得广播变量的相同值（例如，如果变量稍后被运送到新节点），该对象 在广播之后不应被修改。

### 累加器
累加器是只能通过关联和交换操作“添加”的变量，因此可以并行有效地支持。它们可以用来实现计数器（如在MapReduce中）或者和。Spark本身支持数字类型的累加器，程序员可以添加对新类型的支持。

作为用户，您可以创建名称或未命名的累加器。如下图所示，一个命名的累加器（在这种情况下counter）将显示在Web用户界面中修改该累加器的阶段。Spark显示由“任务”表中的任务修改的每个累加器的值。

![Spark UI中的累加器](https://spark.apache.org/docs/latest/img/spark-webui-accumulators.png)

跟踪用户界面中的累加器对于理解运行阶段的进度非常有用（注意：Python尚不支持）。

数值累加器可以通过调用SparkContext.longAccumulator()或SparkContext.doubleAccumulator() 累加Long或Double类型的值来创建。在集群上运行的任务可以使用该add方法添加到集群中。但是，他们无法读懂它的价值。只有驱动程序可以使用其value方法读取累加器的值。

下面的代码显示了一个累加器被用来加总一个数组的元素：

```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```
虽然这段代码使用了对Long类型的累加器的内置支持，程序员也可以通过继承AccumulatorV2来创建它们自己的类型。AccumulatorV2抽象类有几个方法必须覆盖：reset将累加器重置为零，add向累加器中添加另一个值，merge将另一个相同类型的累加器合并到该累加器中 。其他必须被覆盖的方法包含在API文档中。例如，假设我们有一个MyVector表示数学向量的类，我们可以这样写：

```scala
class VectorAccumulatorV2 extends AccumulatorV2[MyVector, MyVector] {

  private val myVector: MyVector = MyVector.createZeroVector

  def reset(): Unit = {
    myVector.reset()
  }

  def add(v: MyVector): Unit = {
    myVector.add(v)
  }
  ...
}

// Then, create an Accumulator of this type:
val myVectorAcc = new VectorAccumulatorV2
// Then, register it into spark context:
sc.register(myVectorAcc, "MyVectorAcc1")
```
请注意，当程序员定义自己的AccumulatorV2类型时，结果类型可能与添加元素的类型不同。

对于仅在动作内执行的累加器更新，Spark保证每个任务对累加器的更新只应用一次，即重新启动的任务不会更新值。在转换中，用户应该意识到，如果任务或作业阶段被重新执行，每个任务的更新可能会被应用多次。

累加器不会改变Spark的懒惰评估模型。如果它们在RDD上的操作中被更新，则其值仅在RDD作为动作的一部分计算之后才被更新。因此，累加器更新不能保证在像lazy这样的惰性转换中执行map()。下面的代码片段演示了这个属性：

```scala
val accum = sc.longAccumulator
data.map { x => accum.add(x); x }
// Here, accum is still 0 because no actions have caused the map operation to be computed.
```

## 部署到群集
在提交申请指南介绍了如何提交申请到集群。简而言之，一旦将应用程序打包为JAR（Java / Scala）或一组.py或多个.zip文件（Python），bin/spark-submit脚本就可以将其提交给任何受支持的集群管理器。

## 从Java / Scala启动Spark作业
该org.apache.spark.launcher 包提供类推出的Spark作为工作使用一个简单的Java API的子进程。

## 单元测试
Spark对任何流行的单元测试框架的单元测试都很友好。只需SparkContext在您的测试中创建一个主URL设置为local，运行您的操作，然后打电话SparkContext.stop()把它撕下来。确保停止finally块或测试框架tearDown方法中的上下文，因为Spark不支持在同一个程序中同时运行的两个上下文。

## 接下来
您可以在Spark网站上看到一些Spark程序示例。另外，Spark在examples目录（Scala， Java， Python， R）中包含了几个示例。您可以通过将类名传递给Spark bin/run-example脚本来运行Java和Scala示例; 例如：

```
./bin/run-example SparkPi
```
对于Python示例，请spark-submit改为使用：
```
./bin/spark-submit examples/src/main/python/pi.py
```
对于R示例，请spark-submit改用：
```
./bin/spark-submit examples/src/main/r/dataframe.R
```
有关优化程序的帮助，配置和 调优指南提供有关最佳实践的信息。它们对于确保您的数据以高效格式存储在内存中尤为重要。有关部署的帮助，群集模式概述描述了分布式操作中涉及的组件以及支持的群集管理器。
