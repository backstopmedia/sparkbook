# Chapter6: Case studies

What's in this chapter?

- Introducing frameworks and libraries that are useful for various kind of use cases
- Providing the area where Spark can be applied to promote your business by using your existing data
- Catch up the future works of Spark

The following code samples have been used in this chapter to describe in a more comprehensive way the topics listed above.

1. Data Warehousing

```bash
$ build/mvn -Pyarn -Phive -Phive-thriftserver \
  -Phadoop-2.6.0 -Dhadoop.version=2.6.0 \
  -DskipTests clean package
```

```bash
./make-distribution.sh --name hadoop2-without-hive \
    —-tgz -Pyarn -Phadoop-2.6 \
    -Pparquet-provided
```

```bash
$ wget http://ftp.yz.yamagata-u.ac.jp/pub/network/apache/hive/hive-1.1.1/apache-hive-1.1.1-bin.tar.gz
$ tar zxvf apache-hive-1.1.1-bin.tar.gz
$ cd apache-hive-1.1.1-bin.tar.gz
$ bin/hive
hive> set spark.home=/location/to/sparkHome;
hive> set hive.execution.engine=spark;
```

2. Machine Learning

## DataFrame

```scala
// it is already defined if you use 
// spark-shell tool.
val sc: SparkContext
// sqlContext is also defined in spark-shell
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
// This is necessary to convert an RDD to a DataFrame
// implicitly
import sqlContext.implicits._
```

```scala

// DataFrame can read a json file without specifying the
// scheme explicitly
val df = sqlContext.read.json(“examples/src/main/resources/people.json")
df.show()
// age  name
// null Michael
// 30   Andy
// 19   Justin
```

```scala

case class Person(name: String, age: Int)
val people = sc.textFile("examples/src/main/resources/people.txt")
  .map(_.split(",")).map(p => Person(p(0), p(1).trim.toInt)).toDF()
people.select(“name”).show()
```

```scala

val people = sc.textFile(“examples/src/main/resources/people.txt") 
import org.apache.spark.sql.type.StructType
import org.apache.spark.sql.type.{StructField,StringType}
// Definition of data schema
val schema = StructType(Seq(StructField("name", StringType, true), 
                            StructField("age", StringType, true)))
// First import as Row class
val rowRDD = people.map(_.split(",")).map(p => Row(p(0), p(1).trim))
// Create DataFrame with the previous definition
val peopleDataFrame = sqlContext.createDataFrame(rowRDD, schema)
```

## MLlib, ML

```scala
import org.apache.spark.ml.classification.LogisticRegression
import org.apache.spark.ml.param.ParamMap
import org.apache.spark.mllib.linalg.{Vector, Vectors}
import org.apache.spark.sql.Row
val pipeline = new Pipeline()
  .setStages(Array(tokenizer, hashingTF, logRegression))
```

```scala
val tokenizer = new Tokenizer()
  .setInputCol(“sentence”).setOutputCol(“words”)
val regexTokenizer = new RegexTokenizer()
  .setInputCol(“sentence”).setOutputCol(“words”)
  .setPattern(“,”) // Tokenize csv like format
val hashingTF = new HashingTF()
  .setInputCol(“words”).setOutputCol(“features”)
```

```scala
val df = sqlContext.createDataFrame(Seq((0, "A"), (1, "B"), (2, "C"), (3, "A"), (4, "A"), (5, "C"))).toDF("id", "category")	
val indexer = new StringIndexer().setInputCol("category").setOutputCol(""categoryIndex")
// Add discrete index as categoryIndex column

val dataset = sqlContext.createDataFrame(Seq((0, 25, 2.0, Vectors.dense(0.0, 10.0, 0.5), 1.0)))
  .toDF("userid", "age", "addressCode", "userFeatures", “admission")
val vectorAssembler = new VectorAssembler().setInputCols(Array("age", "userFeatures"))
  .setOutputCol(“features")
```

```scala
val dt = new DecisionTreeClassifier().setLabelCol(“label”)
  .setFeaturesCol(“features”)
val pipeline = new Pipeline().setStages(Array(firstTransformer,
  secondTransformer, dt)
// Returned value is DecisionTreeClassfierModel
val model = pipeline.fit(dataframe)
```

```scala
val predictions = model.tranform(testData)
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol(“label”).setPredictionCol(“prediction”)
  .setMetricName(“precision”)
```

Grid search

```scala
val paramGrid = new ParamGridBuilder()
  .addGrid(dt.impurity, Array(“entropy”, “gini”))
  .addGrid(dt.minInfoGain, Array(1, 10, 100)).build()
```

Cross validation

```scala
val cv = new CrossValidator().setEstimator(pipeline)
  .setEvaluator(new MultiClassificationEvaluator)
  .setEstimatorParamMaps(paramGrid).setNumFolds(3)
val model = cv.fit(datasets)
```

## Mahout on Spark

```bash
$ cd $SPARK_HOME
# Launch Spark standalone cluster on local machine
# It requires administrative privileges
$ ./sbin/start-all.sh
```

```bash
$ export SPARK_HOME=/path/to/spark/directory
$ export MAHOUT_HOME=/path/to/mahout/directory
# Set the url that you found on Spark UI
$ export MASTER=spark://XXXXXX:7077
```

```bash
$ cd $MAHOUT_HOME
$ bin/mahout spark-shell
```

## Hivemall on Spark

```bash
$ build/mvn -Phive -Pyarn -Phadoop-2.6 \
  -Dhadoop.version=2.6.0 -DskipTests clean package
$ $SPARK_HOME/bin/spark-shell \
  —packages maropu:hivemall-spark:0.0.5
```

```scala
val model = sqlContext.createDataFrame(dataset).train_logregr(add_bias($”features”), $""label”)
  .groupby(“feature”).agg(“weight”,“avg”).as(“feature”, “weight”)
```

3. External Frameworks

spark-jobserver

```bash
$ docker run -d -p 8090:8090 \
  velvia/spark-jobserver:0.5.2-SNAPSHOT
$ git clone https://github.com/spark-jobserver/spark-jobserver.git
$ cd spark-jobserver
$ sbt job-server-tests/package
# You can get build test package as a jar format under 
# job-server-tests/target/scala-2.10/job-server-tests_2.10-0.6.1-SNAPSHOT.jar, though version number
# might be a little bit different
```

```bash
$ curl --data-binary @job-server-tests/target/scala-2.10/job-server-tests_2.10-0.6.1-SNAPSHOT.jar \
    http://<Your Docker Host IP>:8090/jars/test
$ curl ‘http://<Your Docker Host IP>:8090/jars'
{
  "tests": "2015-11-12T02:26:50.069-05:00"
}
```

```bash
$ curl ‘http://<Your Docker Host IP>:8090/jobs’
{
  "duration": "0.448 secs”,
  "classPath": "spark.jobserver.WordCountExample",
  "startTime": “2015-11-12T03:01:12.362-05:00",
  "context": "0a518c58-spark.jobserver.WordCountExample",
  "status": “FINISHED",
  "jobId": “aed9a387-5319-4d8e-ac3d-0f1ce9d4b1a1”
}
```

```bash
$ curl http://<Your Docker Host IP>:8090/jobs/aed9a387-5319-4d8e-ac3d-0f1ce9d4b1a1
{
  "status": "OK",
  "result": {
    "takeshi": 1,
    "nobita": 2,
    "suneo": 2,
    "dora": 1
  }
}
```

4. Future Works

Installing Parameter server

```bash
$ sudo apt-get update && \
  sudo apt-get install -y build-essential git
$ git clone https://github.com/dmlc/ps-lite
$ cd ps-lite && make deps -j4 && make -j4
```

Installing deeplearning4j

```bash
$ git clone git@github.com:deeplearning4j/dl4j-spark-mlexamples.git
$ cd dl4j-spark-ml-examples
$ mvn clean package -Dspark.version=1.5.2 \
  -Dhadoop.version=2.6.0
```

```bash
$ wget http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz
$ gunzip train-images-idx3-ubyte
$ wget http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz
$ gunzip train-labels-idx1-ubyte
$ mv train-images-idx3-ubyte \
  /path/to/dl4j-spark-ml-examples/data
$ mv train-labels-idx1-ubyte \
  /path/to/dl4j-spark-ml-examples/data
$ MASTER=local[4] bin/run-example ml.JavaMnistClassification
```























