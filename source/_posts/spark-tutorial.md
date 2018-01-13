---
title: spark 官方教程
date: 2018-01-12 22:08:50
tags:
categories: spark
---


本教程提供了使用Spark的快速入门。我们将首先通过Spark的交互式shell（Python或Scala）介绍API，然后展示如何用Java，Scala和Python编写应用程序。

要遵循本指南，请先从Spark网站下载Spark的打包版本 。由于我们不会使用HDFS，因此您可以下载任何版本的Hadoop的软件包。

请注意，在Spark 2.0之前，Spark的主要编程接口是弹性分布式数据集（RDD）。在Spark 2.0之后，RDD被数据集取代，数据集类似于RDD类型强大的类型，但在引擎盖下有更丰富的优化。RDD接口仍然受支持，您可以在RDD编程指南中获得更完整的参考资料。但是，我们强烈建议您切换到使用数据集，这比RDD具有更好的性能。请参阅SQL编程指南以获取有关数据集的更多信息。

## 使用Spark Shell进行交互式分析
### 基本
Spark的shell提供了一个学习API的简单方法，也是一个交互式分析数据的强大工具。它可以在Scala（在Java VM上运行，因此是使用现有Java库的好方法）或Python中提供。通过在Spark目录中运行以下代码来启动它：

```
./bin/spark-shell
```

Spark的主要抽象是一个名为Dataset的分布式集合。数据集可以通过Hadoop InputFormats（例如HDFS文件）或通过转换其他数据集来创建。让我们从Spark源目录中的README文件的文本中创建一个新的数据集：

```scala
scala> val textFile = spark.read.textFile("README.md")
textFile: org.apache.spark.sql.Dataset[String] = [value: string]
```

您可以直接从数据集中获取值，通过调用某些操作，或者转换数据集来获取新的值。有关更多详细信息，请阅读API文档。

```scala
scala> textFile.count() // Number of items in this Dataset
res0: Long = 126 // May be different from yours as README.md will change over time, similar to other outputs

scala> textFile.first() // First item in this Dataset
res1: String = # Apache Spark
```

现在让我们转换这个数据集到一个新的。我们调用filter返回一个新的数据集与文件中的项目的一个子集。

```scala
scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
linesWithSpark: org.apache.spark.sql.Dataset[String] = [value: string]
```
我们可以把变革和行动联系在一起：

```scala
scala> textFile.filter(line => line.contains("Spark")).count() // How many lines contain "Spark"?
res3: Long = 15
```

### 更多关于数据集操作
数据集操作和转换可用于更复杂的计算。比方说，我们想找到最多的单词：

```scala
scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
res4: Long = 15
```
这首先将一行映射到一个整数值，创建一个新的数据集。reduce在该数据集上调用以查找最大的字数。参数map和reduce是Scala函数文字（闭包），并可以使用任何语言功能或Scala / Java库。例如，我们可以轻松地调用其他地方声明的函数。我们将使用Math.max()函数来使这个代码更容易理解：

```scala
scala> import java.lang.Math
import java.lang.Math

scala> textFile.map(line => line.split(" ").size).reduce((a, b) => Math.max(a, b))
res5: Int = 15
```
一种常见的数据流模式是MapReduce，正如Hadoop所普及的。Spark可以轻松实现MapReduce流程：

```scala
scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]
```

在这里，我们要求flatMap将一行数据集转换为一个数据集的单词，然后组合groupByKey并count计算文件中每个单词的数量作为（String，Long）对的数据集。为了收集我们shell中的字数，我们可以调用collect：

```scala
scala> wordCounts.collect()
res6: Array[(String, Int)] = Array((means,1), (under,2), (this,3), (Because,1), (Python,2), (agree,1), (cluster.,1), ...)
```

### 缓存
Spark还支持将数据集提取到集群范围内的内存缓存中。当重复访问数据时，如查询小的“热”数据集或运行迭代算法（如PageRank）时，这非常有用。作为一个简单的例子，让我们标记我们的linesWithSpark数据集被缓存：

```scala
scala> linesWithSpark.cache()
res7: linesWithSpark.type = [value: string]

scala> linesWithSpark.count()
res8: Long = 15

scala> linesWithSpark.count()
res9: Long = 15
```

使用Spark探索和缓存100行文本文件似乎很愚蠢。有趣的部分是这些相同的函数可以用在非常大的数据集上，即使它们被划分为数十或数百个节点。您也可以bin/spark-shell按照RDD编程指南中的描述，通过连接到集群来交互地完成此操作。

## 自包含的应用程序
假设我们希望使用Spark API编写一个自包含的应用程序。我们将通过一个简单的应用程序在Scala（与SBT），Java（与Maven）和Python（PIP）。

我们将在Scala中创建一个非常简单的Spark应用程序 - 这么简单，事实上，它被命名为SimpleApp.scala：

```scala
/* SimpleApp.scala */
import org.apache.spark.sql.SparkSession

object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
    val logData = spark.read.textFile(logFile).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println(s"Lines with a: $numAs, Lines with b: $numBs")
    spark.stop()
  }
}
```
请注意，应用程序应该定义一个main()方法而不是扩展scala.App。子类scala.App可能无法正常工作。

这个程序只是计算Spark自述文件中包含'a'的行数和包含'b'的数字。请注意，您需要将YOUR_SPARK_HOME替换为安装Spark的位置。与之前使用Spark shell初始化自己的SparkSession的例子不同，我们初始化一个SparkSession作为程序的一部分。

我们调用SparkSession.builder构造[[SparkSession]]，然后设置应用程序名称，最后调用getOrCreate获取[[SparkSession]]实例。

我们的应用程序依赖于Spark API，所以我们还将包含一个sbt配置文件 build.sbt，它解释了Spark是一个依赖项。该文件还添加了Spark所依赖的存储库：

```
name := "Simple Project"

version := "1.0"

scalaVersion := "2.11.8"

libraryDependencies += "org.apache.spark" %% "spark-sql" % "2.2.1"
```

对于SBT正常工作，我们需要布局SimpleApp.scala并build.sbt 根据典型的目录结构。一旦到位，我们可以创建一个包含应用程序代码的JAR包，然后使用spark-submit脚本来运行我们的程序。

```
# Your directory layout should look like this
$ find .
.
./build.sbt
./src
./src/main
./src/main/scala
./src/main/scala/SimpleApp.scala

# Package a jar containing your application
$ sbt package
...
[info] Packaging {..}/{..}/target/scala-2.11/simple-project_2.11-1.0.jar

# Use spark-submit to run your application
$ YOUR_SPARK_HOME/bin/spark-submit \
  --class "SimpleApp" \
  --master local[4] \
  target/scala-2.11/simple-project_2.11-1.0.jar
...
Lines with a: 46, Lines with b: 23
```

## 更多
祝贺您运行您的第一个Spark应用程序！

有关API的深入概述，请从RDD编程指南和SQL编程指南开始，或者参阅其他组件的“编程指南”菜单。
要在群集上运行应用程序，请转到部署概述。
最后，Spark在examples目录（Scala， Java， Python， R）中包含了几个样本。你可以运行它们如下：

```scala
# For Scala and Java, use run-example:
./bin/run-example SparkPi

# For Python examples, use spark-submit directly:
./bin/spark-submit examples/src/main/python/pi.py

# For R examples, use spark-submit directly:
./bin/spark-submit examples/src/main/r/dataframe.R
```
