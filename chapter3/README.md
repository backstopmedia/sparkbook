# Chapter 3: Performance Tuning

This chapter explores several techniques to improve the performance of a Spark application covering a large set of topics:
  - Spark's execution model
  - The way data is being shuffled across cluster
  - Why partitioning is so important
  - Data serialization
  - Spark cache
  - Memory management
  - Shared variables
  - Data locality

The following code samples have been used in this chapter to describe in a more comprehensive way the topics listed above.

### Spark Execution Model

1. Code sample to showcase some narrow transformations:

        val numbers = sc.parallelize(nrCollection)
        val multiplied = numbers.filter(_%2 == 0).map(_ * 3).collect()

2. Code sample to showcase stage boundary transformations:
    
        val words = sc.textFile("textFilePath").flatMap(_.split(' ')) 
        val wordCounts = words.map((_, 1)).reduceByKey(_ + _)
        val filteredWords = wordCounts.filter(_._2 == 10)
        val characters = filteredWords.flatMap(_._1.toCharArray)
                                .map((_, 1)).reduceByKey(_ + _)
        characters.collect()

### Controlling Parallelism
1. Code sample to control the partition number for an RDD:

        sc.textFile(<inputPath>, <minPartitions>)
        sc.parallelize(<sequence>, <numSlices>)

2. Example of a transformation that causes data shuffling:

        rdd.groupByKey(<numTasks>)

### Partitioners

1. Examples of how to use a custom partitioner (either passing it to the partitionBy function or to the shuffle based functions):

        pairRdd.partitionBy(new MyPartitioner(3))
        firstPairRdd.join(secondPairRdd, new MyPartitioner(3))

### Shuffling and Data Partitioning
      
1. Code sample that joins a partitioned RDD with one that is not partitioned (The partitioned RDD passes the partitioner to the resulted RDD):

        val partitionedRdd = pairRdd.partitionBy(new HashPartitioner(3))
        val joinedRdd = partitionedRdd.join(otherRdd)
        
2. Code sample to showcase a way to avoid data shuffling when joining two RDDs - make use of partitioning:
        
        val partitionedRdd = pairRdd.partitionBy(new HashPartitioner(3))
        val otherRdd = partitionedRdd.mapValues(value => value + 1)
        val joinedRdd = partitionedRdd.join(otherRdd)

### Operators and Shuffling

1. Code sample to back up the reduceByKey vs groupByKey operators comparison:

        val words = Array ("one", "two", "one", "one", "one", "two", "two", "one", "one", "two", "two")
        val wordsPairRdd = sc.parallelize(words).map(w => (w,1))
        
        wordsPairRdd.groupByKey().map(t => (t._1, t._2.sum))
        wordsPairRdd.reduceByKey(_ + _)

2. Code sample to back up the reduceByKey vs aggregateByKey operators comparison:

        val userAccesses = sc.parallelize(Array(("u1", "site1"), ("u1", "site2"), ("u1", "site1"), ("u2", "site3"), ("u2", "site4"), ("u1", "site1")))
        val mapedUserAccess = userAccesses.map(userSite => 
                                    (userSite._1, Set(userSite._2)))
        
        val distinctSites = mapedUserAccess.reduceByKey(_ ++ _)
        val zeroValue = collection.mutable.Set[String]()
        
        val aggredated = userAccesses.aggregateByKey (zeroValue)((set, v) => 
                        set += v, (setOne, setTwo) => setOne ++= setTwo)

### Serialization

1. Code sample to showcase how to set the Kryo serializer:

        val configuration = new SparkConf()
        configuration.set(“spark.serializer”,"org.apache.spark.serializer.KryoSerializer")

2. Examples of how to register your classes to the Kryo serializer (programmatically or in the the spark-defaults.conf file):

        configuration.registerKryoClasses(Array(classOf[CustomClassOne],
                            classOf[CustomClassTwo]))
                            
        spark.kryo.classesToRegister=org.x.CustomClassOne,org.x.CustomClassTwo
        

### Kryo Registrators

1. Code sample to showcase how to specify the order in which the classes will be registered to Kryo:

        kryo.register(classOf[CustomClassOne], 0)
        kryo.register(classOf[CustomClassTwo], 1)
        kryo.register(classOf[CustomClassThree], 2)
        
2. Code sample to showcase how to change the default serializers for the classes registered to Kryo:

        kryo.register(classOf[CustomClassOne], new CustomSerializerOne())
        kryo.register(classOf[CustomClassTwo], new CustomSerializerTwo())
        
### Spark Cache        
        
1. Code samples to exemplify how to persist your RDDs with different storage levels:

        myRdd.cache()
        myRdd.persist()
        myRdd.persist(StorageLevel. MEMORY_ONLY)
        
        myRdd.persist(StorageLevel. MEMORY_ONLY_SER)
        
        myRdd.persist(StorageLevel. MEMORY_AND_DISK)
        
        myRdd.persist(StorageLevel. MEMORY_AND_DISK_SER)
        
        myRdd.persist(StorageLevel.DISK_ONLY)
        
        myRdd.persist(StorageLevel. MEMORY_ONLY_2)
        myRdd.persist(StorageLevel. MEMORY_ONLY_SER_2)
        myRdd.persist(StorageLevel. MEMORY_AND_DISK_2)
        myRdd.persist(StorageLevel. MEMORY_AND_DISK_SER_2)
        myRdd.persist(StorageLevel.DISK_ONLY_2)
        
        myRdd.persist(StorageLevel.OFF_HEAP)
        
2. Code sample to show how to remove an RDD from cache:

        myRdd.unpersist()
    
3. Code sample to show how to change the minimum duration that each DStream should remember its RDDs:

        streamingContext.remember(duration)
        
### Spark SQL cache
    
1. Code samples to show different ways to cache a table:
    
        dataFrame.cache()
        sparkSqlContext.cacheTable("myTableName")
        sparkSqlContext.sql("cache table myTableName")

2. Code sample to show how to use lazy cache:

        sparkSqlContext.sql("cache lazy table myTableName")

3. Code sample to show how to manually remove a table from cache:

        dataFrame.unpersist()
        sparkSqlContext.uncacheTable("myTableName")
        sparkSqlContext.sql("uncache table myTableName")
        
### Shared Variables

1. Code sample to showcase a clojure:

        val power = 2 
        val raiseToPower =  (number : Double) => pow(number, power)
        
2. Code sample to showcase a clojure in Spark:

        import scala.math._
        val myRdd = sc.parallelize(Array(1,2,3,4,5,6,7))
        val power = 2 
        myRdd.map(pow(_,power))
    
### Broadcast Variables

1. Code sample to showcase a clojure in Spark (second example): 

        val myRdd = sc.parallelize(Array(1,2,3,4,5,6,7))
        val myDictionary = Map(1 -> "sentence1", 2 -> "sentence2", ..)
        myRdd.foreach(value => 
            print(myDictionary.getOrElse(value, "Doesn't exist")))
            
2. Code sample to show how broadcast the dictionary from the previous example:

        val myRdd = sc.parallelize(Array(1,2,3,4,5,6,7))
        val myDictionary = Map(1 -> "sentence1", 2 -> "sentence2", ..)
        val broadcastedDict = sc.broadcast(myDictionary) 
        myRdd.foreach(value => print(broadcastedDict.value
                                .getOrElse( value, "Doesn't exist")))

3. Code sample to show how to set the broadcast implementation:

        val configuration = new SparkConf()
        configuration.set("spark.broadcast.factory",
                "org.apache.spark.broadcast.TorrentBroadcastFactory")
        configuration.set("spark.broadcast.blockSize", "4m")
        
### Accumulators

1. Code sample to showcase an incorrect way to use counters:

        val myRdd = sc.parallelize(Array(1,2,3,4,5,6,7))
        var evenNumbersCount = 0
        var unevenNumbersCount = 0 
        myRdd.foreach(element => 
                {if (element % 2 == 0) evenNumbersCount += 1
		         else unevenNumbersCount +=1
                })

2. Code sample to showcase the use of accumulators:

        val myRdd = sc.parallelize(Array(1,2,3,4,5,6,7))
        val evenNumbersCount = sc.accumulator(0, "Even numbers")
        var unevenNumbersCount = sc.accumulator(0, "Uneven numbers")
        myRdd.foreach(element => 
                {if (element % 2 == 0) evenNumbersCount += 1
		        else unevenNumbersCount +=1
                })
        println(s" Even numbers ${ evenNumbersCount.value }")
        println(s" Uneven numbers ${ unevenNumbersCount.value }")
        
3. Code sample to showcase a custom Accumulator implementation:

        object ErrorFilesAccum extends AccumulatorParam[String] {
            def zero(initialValue: String): String = {
                initialValue
             }
            def addInPlace(s1: String, s2: String): String = {
   	            s1+”,”+s2
            }
        }
        
        val errorFilesAccum = sc.accumulator ("","ErrorFiles") (ErrorFilesAccum)
        
### Data locality

1. Code sample to showcase how to configure the amount of time for Spark to wait for an executor to become free for each individual data locality level:

        val configuration = new SparkConf()
        configuration.set("spark.locality.wait", "3s")
        configuration.set("spark.locality.wait.node", "3s")
        configuration.set("spark.locality.wait.process", "3s")
        configuration.set("spark.locality.wait.rack", "3s")
        
