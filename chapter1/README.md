# Chapter 1: Getting Your Spark Job to Finish

What’s in this chapter?

- Installation of the necessary components
 - Distributed computing
 - Understanding resource management
 - Using various formats for storage 
 - Making sense of monitoring and instrumentation
 
The following code samples have been used in this chapter to describe in a more comprehensive way the topics listed above.

### Text Files

1. Code sample to showcase how to process a CSV file creating a DataFrame on top of it:

        import sqlContext.implicits._
        case class Pet(name: String, race : String)
        val textFileRdd = sc.textFile("file.csv”)
        val schemaLine = textFileRdd.first()
        val noHeaderRdd = textFileRdd.filter(line =>!line.equals(schemaLine))
        val petRdd = noHeaderRdd.map(textLine => {
                    val columns = textLine.split(",")
                    Pet(columns(0), columns(1))})
        val petDF = petRdd.toDF()
        
2. Code sample to showcase how to process a CSV file using the spark-csv package:

        val df = sqlContext.read
            .format("com.databricks.spark.csv")
            .option("header", "true")
            .option("inferSchema", "true")
            .load("file.csv")
            
3. Code sample to showcase how to read a JSON file with and without specifying the schema:

        val schema = new StructType(Array(
            new StructField("name", StringType, false),
            new StructField("age", IntegerType, false)))

        val specifiedSchema= sqlContext.jsonFile("file.json",schema)
        val inferedSchema = sqlContext.jsonFile("file.json")
        
### Sequence Files

1. Code sample to showcase how to load a sequence file:

        val seqRdd = sc.sequenceFile("filePath", classOf[Int], classOf[String])

### Avro files

1. Code sample to showcase how to load an avro file:

        import com.databricks.spark.avro._ 
        val avroDF = sqlContext.read.avro("pathToAvroFile")

### Parquet files

1. Code sample to showcase how to load a parquet file having the schema merging enabled:

        val parquetDF = sqlContext.read
                    .option("mergeSchema", "true")
                    .parquet("parquetFolder")
                    
