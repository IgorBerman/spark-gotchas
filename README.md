This document was created as our list of gotchas we've encountered during setuping spark based etl pipeline. We(team of 4) started without any experience with spark except for trivial tutorials from spark site. Experienced spark users might find listed items as obvious, however we still think that many will find it usefull. 

After few weeks of strugling trying to build real pipeline with spark core we started to feel frustration, since everybody talks about spark, everybody thinks it's cool project(but IMHO few really work with it). We just had hard times with it. Afterwards we discovered few such reports, so if you are feeling it too - it's not ONLY you :)

Short description of our project, so you'll understand our usecase: might be for you some/all advices won't be relevant:
- Batch job that ingests raw events on daily basis
- Multitenant environment (i.e. same pipeline for hundrends of customers)
- Many aggregations i.e. many shuffling
- Most of our code is written in java, might be few mentioned things wont apply for scala codebase

So:

1. Tests and automation. I can't stress it more. Many below problems were discovered by unit-testing and integration testing
  - unit tests for functions/composite functions(transformation chains)
  - integration test with embedded/local spark context
  - automation for provisioning servers and deployment(short cycles)
  - check [spark-testing-base](https://github.com/holdenk/spark-testing-base)
2. newAPIHadoopFile + avro(this was our major issue with spark). Below is copy-pasted scaladoc for this method:
  ```
    /**
     * Get an RDD for a given Hadoop file with an arbitrary new API InputFormat
     * and extra configuration options to pass to the input format.
     *
     * '''Note:''' Because Hadoop's RecordReader class re-uses the same Writable object for each
     * record, directly caching the returned RDD will create many references to the same object.
     * If you plan to directly cache Hadoop writable objects, you should first copy them using
     * a `map` function.
     */
    def newAPIHadoopFile[K, V, F <: NewInputFormat[K, V]
  ```
At given point we haven't started to optimize, we needed to get code right and only then wanted to optimize it, so I was thinking to myself: hmmm...no - I don’t cache rdd...and yet: **DEEP-COPY YOUR OBJECTs**...unless you want to spend a week or so trying to understand why your data is corrupted after aggregation(In general hadoop or spark code caches some buffer in avro reader). You can meet this issue in whole different ways.

3. Use logically immutable objects inside RDDs!(probably java only)
  - you still can create new object and then set data field by field
  - inside spark transformation don’t change input object, create new one
  - why? - remember: rdd or part of it can be recomputed due to failure, so your input object might be changed few times

3. Partitioner - it's your best friend for performance
  - map,distinct…. - transformations that lose partitioner, so try to avoid them if you can - you can use mapPartitions, mapValues instead of map or reduceByKey instead of distinct etc.
  - newAPIHadoopFile - reading partitioned rdd from previous batch cycle loses partitioner. Our batch process reads data, aggregates it and saves it back to s3/hdfs. The volume is not few kb. When next day same process reads rdd that were stored with partitioner inside, it's not there anymore! Spark loses the partitioner information.
  - [there is PR](https://github.com/apache/spark/pull/4449) that describes how to solve this problem. We might [release](https://github.com/IgorBerman/DySparkExtensionOS) our internal project in the future that implements standalone extention to spark core for it, in general everything in Imran's PR.

4. Tuning number of partitions -  a most "voodoo" part, however you wont success using spark unless you'll understand how to tune this. There are few factors that you'll need to take into account:
  - Memory pressure and how much memory is available for each core(~ total heap * shuffle or cache fraction / number of cores)
  - Use Spark ui metric: Shuffle spill memory/Shuffle spill disk - if you see it a lot either upgrade your hardware, or make each partition smaller, or increase shuffle fraction
  - ShuffleMemoryManager: Thread 61 waiting for at least 1/2N of shuffle memory pool to be free - same as above
  - Our rule of thumb: 64 MB - we created simple tool that computes number of partitions as function of total size of data and writes it as simple csv file(line per data type per tenant), so we run it once in a while to update number of partitions. Sometimes, when there is huge and unpredicted spike in data volume we experience performance problems.

5. When to use persist():
  - spark 1.4.1+ has nice DAG visualisation - look for same transformation chains
  - when your DAG splits into 2+ branches, but remember that caching adds to memory pressure or takes place on disk
  - dont forget to unpersist(when you can), it will help

6. Caching data in memory vs disk. We rarely cache data in memory(we just don't have so much memory available), so few factors that influence your decisions:
  - size of rdd in memory vs size of rdd on disk(compression) - measure what is your factor between data on disk vs data in memory. Don't forget that not all heap is used for caching only memory fraction
  - spark.shuffle.memoryFraction vs spark.storage.memoryFraction - if you have many shuffles, might be you want to optimize for shuffling and not for caching
  - we prefer to improve shuffle performance
  - and we use ssd disk as tmp storage(spark.local.dir=<path to raid0 of ephemeral ssds>)

7. Kryo
  - **Use it!**
  - register every class
  - find out what to register with integration test with spark.kryo.registrationRequired set to true

8. spark.speculation and missing partitions - we spent a lot of time trying to understand what we were doing wrong
  - https://issues.apache.org/jira/browse/SPARK-4879
  - don’t use speculation when you writing output to s3, hdfs(?)

9. S3a vs S3n
  - move to s3a asap, but to work properly you should
  - upgrade to hadoop 2.7+
  - hdfs is the best, but has limited scalability
  - writing data to s3 is unlimitedly scalable, but has problem with file output commiter: when spark writes each part in parallel it first writes to _temporary directory/bucket and then on commit stage all parts are moved to the destination. While in hdfs "move" is implemented efficiently, in s3 "move" is copying each part 1-by-1(forget parallelism!) to the destination bucket. When your rdd has 256+ parts, this commiting stage will be considerable. Moreover in spark UI you'll see that job is already done, but next job is not starting. To solve this we are testing in production [direct output commiter for avro format](https://github.com/IgorBerman/directavro)(Pay attention to the limitation it imposes on your settings - you can't append data and you can't use speculation mode)

10. StackOverflowException and long transformation chains - sometimes you'll get this error and won't understand what's wrong. The problem is that your transformation chain is too long to be serialized, so 
  - it usually happens when you have some sort of iterative transformations
  - there are few links in internet that can help you using checkpoint, however you'll get performance degradation
  - our advice - rethink your approach - reduce depth(chain to tree, combine several iterative steps into one)

11. Job submission pools - probably will be relevant only in multitenant environments, when you have different sizes of customers
  - small tenants with small data - pool will help you to utilize all cluster cores(e.g. I have 32 cores and there are many small customers with data of 4 partitions only, so to fully utilize my cluster I'll need thread pool of size 8, that will use same spark context and will cause up to 8 jobs to be executed in parallel)
  - currently we assign customers into 2 groups and run our pipeline with different settings for each group

12. Flow/Jobs management tool
  - Tried Spotify's Luigi - not good for spark in multitenant environment - every job is submitted as a new spark context - big overhead
  - internally we implemented brother of Luigi - Mario
  - You can try JobServer too
  - NIH, enough time spent on Luigi
  - heavily based on Luigi design
  - written in Java as our primary language(Spotify seems to move to other tools too…)
  - supports multitenant env
  - has basic UI
  - in general few core classes and a lot of interfaces that you should implement by yourself(for monitoring, for errors reporting, what is your job, how it’s identified etc)
  - might be open sourced - time priorities

14. Other tweeking:
  - spark.rdd.compress=true
  - spark.shuffle.consolidateFiles=true
