# README file for Kafka-Spark-Consumer

NOTE : This Kafka Spark Consumer code is taken from Kafka spout of the Apache Storm project (https://github.com/apache/storm/tree/master/external/storm-kafka), 
which was originally created by wurstmeister (https://github.com/wurstmeister/storm-kafka-0.8-plus).
Original Storm Kafka Spout Code has been modified to work with Spark Streaming.

This utility will help to pull messages from Kafka using Spark Streaming and have better handling of the Kafka Offsets and handle failures.

This Consumer have implemented a Custom Reliable Receiver which uses low level Kafka Consumer API to fetch messages from Kafka and store every received block in Spark BlockManager. The logic will automatically detect number of partitions for a topic and spawn as many Kafka Receiver based on configured number of Receivers. Each Receiver can fetch messages from one or more Kafka Partitions.  Receiver commit the Kafka offsets of Received blocks once store to Spark BlockManager is successful.

e.g. if Kafka have 100 partitions of a Topic, and Spark Consumer if configured with 20 Receivers, each Receiver will handle 5 partition. 

In Spark driver code , Receivers is launched by calling **ReceiverLauncher.launch**

Please see Java or Scala code example on how to use this Low Level Consumer

## Salient Feature of Kafka-Spark-Consumer

- This Consumer uses **Zookeeper** for storing the **consumed** and **processed** offset for each Kafka partition, which will help to recover in case of failure
- Spark streaming job using this Consumer does not require **WAL** for recovery from Driver or Executor failures. As this consumer has capability to store the processed offset after every Batch interval, in case of any failure, Consumer can start from last **Processed** offset.
- This Consumer has implemented **PID (Proportional , Integral , Derivative )**  based Rate Controller for controlling Back-Pressure by altering the **Spark Block Size**
- This consumer have capability to use **Message Interceptor** which can be used to preprocess kafka messages before writing to Spark Block Manager

# What is Different from Spark Out of Box Kafka Consumers

* This Consumer is Receiver based fault tolerant reliable consumer designed using Kafka Low Level Simple Consumer API. This Receiver is designed to recover from any underlying failure and does not require any WAL feature in case of Driver failure. Please refer to **Consumer Recovery from Driver/Executor Crash** section below for more details. 

* This consumer uses Kafka Low Level Simple Consumer API which is much faster than Kafka High Level API. Low Level Consumer API is compatible across Kafka versions 0.8, 0.9 and 0.10

* This Consumer have mechanism to create Block from Kafka Stream and write to Spark BlockManager ( See more details in **Consumer Tuning Options** section below ).  

* This Consumer has in-built PID (Proportional, Integral, Derivative ) Controller to control the **Spark Back Pressure** by modifying the size of Block it can consume. The PID Controller rate feedback loop mechanism is built using Zookeeper. The logic to control Back Pressure is by altering size of the Block consumed per batch from Kafka. As the Back Pressure is built into the Consumer, this consumer can be used with any version of Spark if there is a need to have a back pressure controlling mechanism in given Spark / Kafka environment. Please refer **Spark Consumer Properties** section on how to enable back pressure. Also see **Consumer Tuning Options**  section on how to tune PID Controller.

* Number of partitions in RDD generated by this consumer is decoupled from the number of Kafka partitions. One can control the RDD partitions by controlling the Block creation interval. Hence the processing parallelism is not dependent on Kafka Partition. Let assume you have Kafka Topic with 10 Partition. And your Block Interval is 200 Ms and Batch Interval is 5 Sec. This Consumer will generate 5 Blocks every second (1 second / Block Interval ) for each Partitions , and 5 x 10 x 5 = 250 Blocks for every Batch. As every block written to Spark BlockManager within Batch interval creates one Partition for underlying RDD , which mean every RDD created per batch will have 250 Partitions and this will increase processing parallelism. Whereas , if RDD partition is same as Kafka partition , every RDD will only have 10 partitions (same as kafka topic partition) and limit your processing parallelism.

* This Consumer will enable end to end **No Data Loss guarantee** without support for Spark WAL feature. Refer to **Consumer Recovery from Driver/Executor Crash** section for more details.

# Instructions for Manual build 

	git clone https://github.com/dibbhatt/kafka-spark-consumer

	cd kafka-spark-consumer

	mvn install

And Use Below Dependency in your Maven 

		<dependency>
				<groupId>kafka.spark.consumer</groupId>
				<artifactId>kafka-spark-consumer</artifactId>
				<version>1.0.8</version>
		</dependency>

# Accessing from Spark Packages

**This Consumer is now part of Spark Packages** : http://spark-packages.org/package/dibbhatt/kafka-spark-consumer

Include this package in your Spark Applications using:

* spark-shell, pyspark, or spark-submit
	$SPARK_HOME/bin/spark-shell --packages dibbhatt:kafka-spark-consumer:1.0.8
* sbt

If you use the sbt-spark-package plugin, in your sbt build file, add:

	spDependencies += "dibbhatt/kafka-spark-consumer:1.0.8"

Otherwise,

	resolvers += "Spark Packages Repo" at "http://dl.bintray.com/spark-packages/maven"
			  
	libraryDependencies += "dibbhatt" % "kafka-spark-consumer" % "1.0.8"


* Maven

In your pom.xml, add:

	<dependencies>
	  <!-- list of dependencies -->
	  <dependency>
		<groupId>dibbhatt</groupId>
		<artifactId>kafka-spark-consumer</artifactId>
		<version>1.0.8</version>
	  </dependency>
	</dependencies>
	<repositories>
	  <!-- list of other repositories -->
	  <repository>
		<id>SparkPackagesRepo</id>
		<url>http://dl.bintray.com/spark-packages/maven</url>
	  </repository>
	</repositories>

		
# Spark Consumer Properties
				
These are the Consumer Properties need to be used in your Driver Code. ( See Java and Scala Code example on how to use these properties)

* ZK quorum details of Kafka Cluster.
	* **zookeeper.hosts**=host1,host2
* Kafka ZK Port
	* **zookeeper.port**=2181
* Kafka Broker path in ZK. Default Kafka settings is /brokers
	* **zookeeper.broker.path**=/brokers
* Kafka Topic to consume
	* **kafka.topic**=topic-name
* Consumer ZK quorum details. Used to store the consumed offset.
	* **zookeeper.consumer.connection**=host1:2181,host2:2181
* ZK Path for storing Kafka Consumer offset. Path will be automatically created
	* **zookeeper.consumer.path**=/spark-kafka
* Kafka Consumer ID. Identifier of the Consumer
	* **kafka.consumer.id**=consumer-id
* OPTIONAL - Force From Start . Default Consumer Starts from Latest offset.
	* **consumer.forcefromstart**=true
* OPTIONAL - Fetch Size in Bytes . Default 512 KB
	* **consumer.fetchsizebytes**=1048576
* OPTIONAL - Fill Frequence in MS . Default 200 milliseconds
	* **consumer.fillfreqms**=250
* OPTIONAL - Consumer Back Pressure Support. Default is false
	* **consumer.backpressure.enabled**=true
* OPTIONAL - Kafka Message Handler. Implementation of KafkaMessageHandler Interface
	* **kafka.message.handler.class**=consumer.kafka.IdentityMessageHandler
* OPTIONAL - Minimum Fetch Size if Back Pressure kicked in. Default 1 KB
	* **consumer.min.fetchsizebytes**=10240
* OPTIONAL - This can further control RDD Partitions. Number of Blocks fetched from Kafka to merge before writing to Spark Block Manager. Default is 1
	* **consumer.num_fetch_to_buffer**=2

# Java Example

    Properties props = new Properties();
    props.put("zookeeper.hosts", "x.x.x.x");
    props.put("zookeeper.port", "2181");
    props.put("zookeeper.broker.path", "/brokers");
    props.put("kafka.topic", "mytopic");
    props.put("kafka.consumer.id", "kafka-consumer");
    props.put("zookeeper.consumer.connection", "x.x.x.x:2181");
    props.put("zookeeper.consumer.path", "/spark-kafka");
    // Optional Properties
    props.put("consumer.forcefromstart", "false");
    props.put("consumer.fetchsizebytes", "1048576");
    props.put("consumer.fillfreqms", "1000");
    props.put("consumer.backpressure.enabled", "true");
    props.put("consumer.num_fetch_to_buffer", "1");
    props.put("kafka.message.handler.class",
          "consumer.kafka.IdentityMessageHandler");

    SparkConf _sparkConf = new SparkConf();
    JavaStreamingContext jsc = new JavaStreamingContext(_sparkConf, Durations.seconds(30));
    // Specify number of Receivers you need.
    int numberOfReceivers = 1;

    JavaDStream<MessageAndMetadata> unionStreams = ReceiverLauncher.launch(
        jsc, props, numberOfReceivers, StorageLevel.MEMORY_ONLY());

    //Get the Max offset from each RDD Partitions. Each RDD Partition belongs to One Kafka Partition
    JavaPairDStream<Integer, Iterable<Long>> partitonOffset = ProcessedOffsetManager
        .getPartitionOffset(unionStreams);

    //Start Application Logic
    unionStreams.foreachRDD(new Function<JavaRDD<MessageAndMetadata>, Void>() {
      @Override
      public Void call(JavaRDD<MessageAndMetadata> rdd) throws Exception {
        List<MessageAndMetadata> rddList = rdd.collect();
        System.out.println(" Number of records in this batch " + rddList.size());
        return null;
      }
    });
    //End Application Logic

    //Persists the Max Offset of given Kafka Partition to ZK
    ProcessedOffsetManager.persists(partitonOffset, props);
    jsc.start();
    jsc.awaitTermination();

Complete example is available here : 

The src/main/java/consumer/kafka/client/SampleConsumer.java is the sample Java code which uses this ReceiverLauncher to generate DStreams from Kafka and apply a Output operation for every messages of the RDD.

		
# Scala Example

    val conf = new SparkConf()
      .setMaster("spark://x.x.x.x:7077")
      .setAppName("LowLevelKafkaConsumer")
      .set("spark.executor.memory", "1g")
      .set("spark.rdd.compress","true")
      .set("spark.storage.memoryFraction", "1")
      .set("spark.streaming.unpersist", "true")

    val sc = new SparkContext(conf)

    //Might want to uncomment and add the jars if you are running on standalone mode.
    sc.addJar("/home/kafka-spark-consumer/target/kafka-spark-consumer-1.0.8-jar-with-dependencies.jar")
    val ssc = new StreamingContext(sc, Seconds(10))

    val topic = "mytopic"
    val zkhosts = "x.x.x.x"
    val zkports = "2181"
    val brokerPath = "/brokers"
    
    //Specify number of Receivers you need. 
    val numberOfReceivers = 1
    val kafkaProperties: Map[String, String] = 
	Map("zookeeper.hosts" -> zkhosts,
        "zookeeper.port" -> zkports,
        "zookeeper.broker.path" -> brokerPath ,
        "kafka.topic" -> topic,
        "zookeeper.consumer.connection" -> "x.x.x.x:2181",
        "zookeeper.consumer.path" -> "/spark-kafka",
        "kafka.consumer.id" -> "kafka-consumer",
        //optional properties
        "consumer.forcefromstart" -> "true",
        "consumer.backpressure.enabled" -> "true",
        "consumer.fetchsizebytes" -> "1048576",
        "consumer.fillfreqms" -> "250",
        "consumer.num_fetch_to_buffer" -> "1")

    val props = new java.util.Properties()
    kafkaProperties foreach { case (key,value) => props.put(key, value)}

    val tmp_stream = ReceiverLauncher.launch(ssc, props, numberOfReceivers,StorageLevel.MEMORY_ONLY)
    //Get the Max offset from each RDD Partitions. Each RDD Partition belongs to One Kafka Partition
    val partitonOffset_stream = ProcessedOffsetManager.getPartitionOffset(tmp_stream)

    //Start Application Logic
    tmp_stream.foreachRDD(rdd => {
        println("\n\nNumber of records in this batch : " + rdd.count())
    } )
    //End Application Logic

    //Persists the Max Offset of given Kafka Partition to ZK
    ProcessedOffsetManager.persists(partitonOffset_stream, props)
    ssc.start()
    ssc.awaitTermination()
	
Complete example is available here :
	
examples/scala/LowLevelKafkaConsumer.scala is a sample scala code on how to use this utility.

# Consumer Recovery from Driver/Executor Crash without WAL

Please refer to this blog which explains why WAL is needed for Zero Data Loss.

https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html

Primary reason for WAL is , Receiver commit offset of consumed block to ZK after same is written to Spark BlockManager ( and may be replicated). If blocks which are already consumed and committed is not processed , Receiver can have data loss. This is because of how Spark applications operate in a distributed manner. When the driver process fails, all the executors running in a standalone/yarn/mesos cluster are killed as well, along with any data in their memory. In case of Spark Streaming, all the data received from sources like Kafka are buffered in the memory of the executors until their processing has completed. This buffered data cannot be recovered even if the driver is restarted.

Hence there is a need for the WAL feature to recover already Received (but not processed) blocks from WAL written to persistence store like HDFS.

But this Consumer has a different mechanism for Driver failure. Along with **consumed** offset of the Received blocks, this Consumer also maintains the offset of the **processed** blocks. Which mean, this consumer can commit offsets of the already processed blocks to ZK, and in case of Driver failures , it can start from last processed offset ( instead last received offset) for every Kafka partition. Thus Consumer can start from exact same offset since the last successful batch was processed and hence No data loss.

## How This Works

Receiver receives a Block of Data equal to configurable FetchSize from Kafka every FillFrequency. A given Block fetched from Kafka may contain many messages. Every Receiver spawns dedicated thread for every Kafka partition. e.g. if there is 5 Receiver for 20 Kafka Partition, each Receiver will spawn 4 threads. Each thread fetch messages from one and only one Kafka psartition and writes Blocks of data to Spark BlockManager.

Every write to Spark Block Manager creates one Partition for underlying RDD. Say for given Batch Interval if there are N blocks written by all Receiving Threads, the RDD for that batch will have N Partition . 

Receiver can write One Block of data pulled from Kafka during every Fetch, or can merge multiple Fetches together . This can be used to further control the RDD partitions. This is controlled by **consumer.num_fetch_to_buffer** property ( default is 1). Receiver wraps every messages of a given Block with some additional MetaData like message offset and kafka Partition ID. 

As every Receiver thread fetch from single Kafka partition, blocks written by any given Receiver thread will contains the messages (and its MetaData) from same Kafka partition. Which mean every partition of a RDD will have messages belongs to single Kafka partition . Also as the messages in given Block are always in ascending order, if you get the highest offset of a given RDD partition,  that will be the highest offset for a Kafka partition for the same RDD partition. 

As one Receiver thread can write multiple blocks to BlockManager, we need to find highest offset for every RDD Partition which belongs to same Kafka partition , and repeat the same for all Kafka partition . Finally we can find the highest offset for a Kafka partition amongst all RDD partition.
e.g. , if RDD Partition 4, 8 and 12 are generated by Receiver Thread X for Kafka Partition Y , and highest offset for 4 is 100, 8 is 400 and 12 is 800;  then highest offset for Kafka Partition  Y for this Batch is 800. 

This Consumer perform very simple Map Reduce logic to get the highest offset for every Kafka partitions belongs to a given RDD for a Batch. This <Partition, Offset> tuple is written back to ZK as **processed** offset after the Batch completes.  

This require following few lines to be added in Spark Driver Code to avail this feature. 


## How to enable Consumer Recovery from Driver Crash

if you see client/SampleConsumer.java or examples/scala/LowLevelKafkaConsumer.scala , you need to add couple of lines (marked as **1** & **2**) in you Driver Code
  
For Java
  
    //Get the stream
    JavaDStream<MessageAndMetadata> unionStreams = ReceiverLauncher.launch(jsc, props, numberOfReceivers, StorageLevel.MEMORY_ONLY())
    
    // 1. Get the Max offset list for each Kafka partitions
    JavaPairDStream<Integer, Iterable<Long>> partitonOffset =  ProcessedOffsetManager.getPartitionOffset(unionStreams);
  
    //Start Application Logic
      process unionStreams
    //End Application Logic

    // 2. Find the Max amongst the list and Persists the Max Offset to ZK
    ProcessedOffsetManager.persists(partitonOffset, props)

For Scala
    
    //Get the Stream 
    val unionStreams = ReceiverLauncher.launch(ssc, props, numberOfReceivers,StorageLevel.MEMORY_ONLY)
    // 1. Get the Max offset list for each Kafka partitions
    val partitonOffset_stream = ProcessedOffsetManager.getPartitionOffset(unionStreams)
    
    //Start Application Logic
      process unionStreams
    //End Application Logic
    
    // 2. Find the Max amongst the list and Persists the Max Offset to ZK
    ProcessedOffsetManager.persists(partitonOffset_stream, props)

# Consumer Tuning Options

## Block Size Tuning :

The Low Level Kafka Consumer consumes messages from Kafka in Rate Limiting way. Default settings can be found in consumer.kafka.KafkaConfig.java class

You can see following two variables

	public int _fetchSizeBytes = 512 * 1024;
	public int _fillFreqMs = 200 ;
	
This suggests that, Receiver for any given Partition of a Topic will pull 512 KB Block of data at every 200ms.
With this default settings, let assume your Kafka Topic have 5 partitions, and your Spark Batch Duration is say 10 Seconds, this Consumer will pull

512 KB x ( 10 seconds / 200 ms ) x 5 = 128 MB of data for every Batch.

If you need higher rate, you can increase the _fetchSizeBytes (via **consumer.fetchsizebytes** property) , or if you need less number of Block generated you can increase _fillFreqMs ( via **consumer.fillfreqms** property).

These two parameter need to be carefully tuned keeping in mind your downstream processing rate and your memory settings.


## Back-Pressure Rate Tuning

You can enable the BackPressure mechanism by setting *consumer.backpressure.enabled* to "true" in Properties used for ReceiverLauncher

The Default PID settings is as below.

* Proportional = 0.75
* Integral = 0.15
* Derivative = 0

If you increase any or all of these , your damping factor will be higher. So if you want to lower the Consumer rate than what is being calculated with default PID settings , you can increase these values.

You can control the PID values by settings the Properties below.

* **consumer.backpressure.proportional**
* **consumer.backpressure.integral**
* **consumer.backpressure.derivative**


# Running Spark Kafka Consumer

Let assume your Driver code is in xyz.jar which is built using the spark-kafka-consumer as dependency.

Launch this using spark-submit

./bin/spark-submit --class x.y.z.YourDriver --master spark://x.x.x.x:7077 --executor-memory 1G /<Path_To>/xyz.jar

This will start the Spark Receiver and Fetch Kafka Messages for every partition of the given topic and generates the DStream.

e.g. to Test Consumer provided in the package with your Kafka settings please modify it to point to your Kafka and use below command for spark submit. You may need to change the Spark-Version and Kafka-Version in pom.xml.

./bin/spark-submit --class consumer.kafka.client.SampleConsumer --master spark://x.x.x.x:7077 --executor-memory 1G /<Path_To>/kafka-spark-consumer-1.0.8-jar-with-dependencies.jar



 
