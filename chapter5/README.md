# sparkbook

## Chapter 5: Fault Tolerance & Job Execution source code

What’s in this chapter?

* Understanding the lifecycle of a Spark application
* Methods of job scheduling with specifics into scheduling from an existing application
* Concepts of fault tolerance and how to keep long running Spark jobs alive

### Fair Scheduler XML Configuration file:

<?xml version=”1.0”?>
<allocations>
  <!-- high priority queue -->
  <pool name=”high-priority”>
    <schedulingMode>FAIR</schedulingMode>
    <weight>4</weight>
    <minShare>64</minShare>
  </pool>

  <!-- medium priority queue -->
  <pool name=”medium-priority”>
    <schedulingMode>FAIR</schedulingMode>
    <weight>2</weight>
    <minShare>16</minShare>
  </pool>

  <!-- low priority queue -->
  <pool name=”low-priority”>
    <schedulingMode>FIFO</schedulingMode>
  </pool>
</allocations>

### You can enable the given pool as follows:

// assuming the Spark Context is `sc`
sc.setLocalProperty(“spark.scheduler.pool”, “high-priority”)
...
<work for the high priority pool>
...
sc.setLocalProperty(“spark.scheduler.pool”, “low-priority”)
...
<work for the low priority pool>
...
// reset the pool back to the default
sc.setLocalProperty(“spark.scheduler.pool”, null)

### A scenario with a time-critical machine learning application. 

1. The model for this application must be re-trained every ten minutes and redeployed. It leverages an ensemble model technique, roughly meaning that the end model is built on a culmination of a few models. Understanding some general parallelism concepts we realize that, because this is an ensemble model, we can train each sub-model independently of each other. Here are the general steps to look at before we implement parallel scheduling:

import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.ml.classification.LogisticRegression
import org.apache.spark.mllib.linalg.{Vector, Vectors}
import org.apache.spark.sql.Row
import org.apache.spark.sql.SQLContext

val conf = new SparkConf()
val sc = new SparkContext(conf)
val sqlContext = new SQLContext(sc)

val train1 = sqlContext.createDataFrame(Seq(
   (1.0, Vectors.dense(0.4, 4.3, -3.4)),
   (1.0, Vectors.dense(1.2, 9.8, -9.5)),
   (0.0, Vectors.dense(-0.1, 12.4, -2.3)),
   (0.0, Vectors.dense(-1.9, 8.7, -4.6))
)).toDF(“label”, “features”)

val train2 = sqlContext.createDataFrame(Seq(
   (0.0, Vectors.dense(0.3, 4.5, 10.1)),
   (1.0, Vectors.dense(3.2, 0.0, -6.3)),
   (1.0, Vectors.dense(0.2, -8.6, 5.4)),
   (1.0, Vectors.dense(0.1, 6.1, -4.5))
)).toDF(“label”, “features”)

val test = sqlContext.createDataFrame(Seq(
   (0.0, Vectors.dense(-0.2, 9.3, 0.9)),
   (1.0, Vectors.dense(1.1, 6.6, -0.4)),
   (1.0, Vectors.dense(3.8, 12.7, 2.0))
)).toDF("label", "features")

val lr = new LogisticRegression()

val model1 = lr.fit(train1)
val model2 = lr.fit(train2)

model1.transform(test)
   .select(“features”, “label”, “probability”, “prediction”)
   .collect()
   .foreach { case Row(features: Vector, label: Double, prob: Vector, prediction: Double) =>
     println(s”Features: $features, Label: $label => Probability: $prob, Prediction: $prediction”)
   }

model2.transform(test)
   .select(“features”, “label”, “probability”, “prediction”)
   .collect()
   .foreach { case Row(features: Vector, label: Double, prob: Vector, prediction: Double) =>
     println(s”Features: $features, Label: $label => Probability: $prob, Prediction: $prediction”)
   }

### Parallel scheduling in Spark

Let's take the same example and apply the technique of parallel scheduling within a Spark application. Here we will construct multiple `SparkContext` objects and call them asynchronously to train each model simultaneously:

import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.ml.classification.LogisticRegression
import org.apache.spark.mllib.linalg.{Vector, Vectors}
import org.apache.spark.sql.Row
import org.apache.spark.sql.SQLContext

val conf = new SparkConf()
val lr = new LogisticRegression()

val test = sqlContext.createDataFrame(Seq(
   (0.0, Vectors.dense(-0.2, 9.3, 0.9)),
   (1.0, Vectors.dense(1.1, 6.6, -0.4)),
   (1.0, Vectors.dense(3.8, 12.7, 2.0))
)).toDF("label", "features")

val modelExec1 = new Thread(new Runnable {
   def run() {
     val sc = new SparkContext(conf)
     val sqlContext = new SQLContext(sc)

     val train = sqlContext.createDataFrame(Seq(
         (1.0, Vectors.dense(0.4, 4.3, -3.4)),
         (1.0, Vectors.dense(1.2, 9.8, -9.5)),
         (0.0, Vectors.dense(-0.1, 12.4, -2.3)),
         (0.0, Vectors.dense(-1.9, 8.7, -4.6))
     )).toDF(“label”, “features”)
     
     val model = lr.fit(train)

     model.transform(test)
         .select(“features”, “label”, “probability”, “prediction”)
         .collect()
         .foreach { case Row(features: Vector, label: Double, prob: Vector, prediction: Double) =>
           println(s”Features: $features, Label: $label => Probability: $prob, Prediction: $prediction”)
     }     
   }
})

val modelExec2 = new Thread(new Runnable {
   def run() {
     ...
     val train = sqlContext.createDataFrame(...)

     val model = lr.fit(train)

     model.transform(test)...
   }
})

modelExec1.start()
modelExec2.start()

### Resilient Distributed Datasets 

1. Because an RDD is only realized on certain commands it needs to track the lineage of commands placed upon it to retrace in the case of an error. An example of a lineage would be the following code:

scala> val rdd = sc.textFile("<some-text-file>")
rdd: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[1] at textFile at <console>:21

scala> rdd.toDebugString
res0: String =
(2) MapPartitionsRDD[1] at textFile at <console>:21 []
 |  spark-join.spark HadoopRDD[0] at textFile at <console>:21 []

scala> val mappedRdd = rdd.map(line => line++"")
mappedRdd: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[2] at map at <console>:23

scala> mappedRdd.toDebugString
res1: String =
(2) MapPartitionsRDD[2] at map at <console>:23 []
 |  MapPartitionsRDD[1] at textFile at <console>:21 []
 |  spark-join.spark HadoopRDD[0] at textFile at <console>:21 []

2. Some examples of API commands that realize these RDD’s are as follows:

scala> val rdd = sc.textFile("sample.txt")
rdd: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[1] at textFile at <console>:21

scala> rdd.count
res0: Long = 7

scala> rdd.collect
res1: Array[String] = Array(1 2 3 4, 2 3 4 5, 3 4 5 6, 4 5 6 7, 5 6 7 8, 6 7 8 9, 7 8 9 0)

scala> rdd.saveAsTextFile("out.txt")

3. Additional to their ability to track lineage RDD’s can checkpoint that lineage to disk. This acts as a metaphorical “save point” for the data making it much easier to recompute failures at the task level which is especially vital for long running Spark applications or applications working iteratively through large amounts of data. To gain the benefits of checkpointing refer to this code sample:

scala> sc.setCheckpointDir("checkpoint/")

scala> val rdd = sc.textFile("sample.txt")
rdd: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[20] at textFile at <console>:21

scala> val mappedRdd = rdd.map(line => line.split(" "))
mappedRdd: org.apache.spark.rdd.RDD[Array[String]] = MapPartitionsRDD[21] at map at <console>:23

scala> mappedRdd.collect
res14: Array[Array[String]] = Array(Array(1, 2, 3, 4), Array(2, 3, 4, 5), Array(3, 4, 5, 6), Array(4, 5, 6, 7), Array(5, 6, 7, 8), Array(6, 7, 8, 9), Array(7, 8, 9, 0))

scala> val stringRdd = mappedRdd.map(a => a.toSet)
stringRdd: org.apache.spark.rdd.RDD[scala.collection.immutable.Set[String]] = MapPartitionsRDD[22] at map at <console>:25

scala> stringRdd.toDebugString
res15: String =
(2) MapPartitionsRDD[22] at map at <console>:25 []
 |  MapPartitionsRDD[21] at map at <console>:23 []
 |  MapPartitionsRDD[20] at textFile at <console>:21 []
 |  sample.txt HadoopRDD[19] at textFile at <console>:21 []

scala> stringRdd.checkpoint

scala> exit
warning: there were 1 deprecation warning(s); re-run with -deprecation for details

$ ls checkpoint/
29f75822-99dd-47ba-b9a4-5a36165e8885

4. One final piece of resiliency baked into the RDD paradigm is that it can seamlessly fall back to leveraging the disk for partitions that are too large to fit into memory. You can easily set this on your RDD with code such as: 

scala> import org.apache.spark.storage.StorageLevel
import org.apache.spark.storage.StorageLevel

scala> val rdd = sc.textFile("sample.txt")
rdd: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[24] at textFile at <console>:24

scala> rdd.collect
res13: Array[String] = Array(1 2 3 4, 2 3 4 5, 3 4 5 6, 4 5 6 7, 5 6 7 8, 6 7 8 9, 7 8 9 0)

scala> rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
res14: rdd.type = MapPartitionsRDD[24] at textFile at <console>:24

### RDD Persistence Reassignment

1. When speaking of RDD persistence and the recommended storage it is helpful to note that odd errors can arise when attempting this for the first time – here we are discussing the issue of persistence reassignment. This can be a common issue when first dealing with RDD persistence and is one of the most common issues to fix. This is caused by an RDD having already been set with a given persistence level and then attempted to be reassigned. The following code snippet will demonstrate:

scala> import org.apache.spark.storage.StorageLevel
import org.apache.spark.storage.StorageLevel

scala> val rdd = sc.parallelize(Seq(Seq(1,2,3),Seq(2,3,4),Seq(3,4,5),Seq(4,5,6)))
rdd: org.apache.spark.rdd.RDD[Seq[Int]] = ParallelCollectionRDD[31] at parallelize at <console>:28

scala> rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
res15: rdd.type = ParallelCollectionRDD[31] at parallelize at <console>:28

scala> rdd.persist(StorageLevel.MEMORY_ONLY)
java.lang.UnsupportedOperationException: Cannot change storage level of an RDD after it was already assigned a level

2. The only resolution is to stop reassigning the persistence level or create a new RDD object with which one can assign a new persistence level to. An example is as follows:

scala> import org.apache.spark.storage.StorageLevel
import org.apache.spark.storage.StorageLevel

scala> val rdd = sc.parallelize(Seq(Seq(1,2,3),Seq(2,3,4),Seq(3,4,5),Seq(4,5,6)))
rdd: org.apache.spark.rdd.RDD[Seq[Int]] = ParallelCollectionRDD[0] at parallelize at <console>:22

scala> rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
res0: rdd.type = ParallelCollectionRDD[0] at parallelize at <console>:22

scala> val mappedRdd = rdd.map(line => line.toSet)
mappedRdd: org.apache.spark.rdd.RDD[scala.collection.immutable.Set[Int]] = MapPartitionsRDD[1] at map at <console>:24

scala> mappedRdd.persist(StorageLevel.MEMORY_ONLY)
res1: mappedRdd.type = MapPartitionsRDD[1] at map at <console>:24

### Batch Versus Streaming

1. To enable write-ahead logs see the following code example:

import org.apache.spark._
import org.apache.spark.streaming._

val conf = SparkConf()
    .setMaster(“local[4]”)
    .setAppName(“<your-app-name-here>”)
    .set("spark.streaming.receiver.writeAheadLog.enable", "true")
    .set("spark.streaming.receiver.writeAheadLog.rollingIntervalSecs", "1")
val sc = new SparkContext(conf)
val ssc = new StreamingContext(sc, Seconds(1))









