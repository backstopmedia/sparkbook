# Chapter 3: Performance Tuning

This chapter explores several techniques to improve the performance of a Spark application covering a large set of topics:
  - Spark's execution model
  - The way data is being shuflled across cluster
  - Why partitioning is so important
  - Data serialization
  - Spark cache
  - Memory management
  - Shared variables
  - Data locality

The following code samples have been used in this chapter to describe in a more comprehensive way the topics listed above.

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

3. Code sample to control the partition number for an RDD:

        sc.textFile(<inputPath>, <minPartitions>)
        sc.parallelize(<sequence>, <numSlices>)

4. Example of a transformation that causes data shuffling:

        rdd.groupByKey(<numTasks>)

5. Examples of how to use a custom partitioner (either passing it to the partitionBy function or to the shuffle based functions):

        pairRdd.partitionBy(new MyPartitioner(3))
        firstPairRdd.join(secondPairRdd, new MyPartitioner(3))
        
6. Code sample that joins a partitioned RDD with one that is not partitioned (The partitioned RDD passes the partitioner to the resulted RDD):

        val partitionedRdd = pairRdd.partitionBy(new HashPartitioner(3))
        val joinedRdd = partitionedRdd.join(otherRdd)
