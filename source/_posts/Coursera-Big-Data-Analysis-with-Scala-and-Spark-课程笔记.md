---
title: '[Coursera] Big Data Analysis with Scala and Spark 课程笔记'
date: 2018-07-15 22:38:38
tags: [Study Notes, Scala, Functional Programming]
categories:
- [Open Course, Coursera]
---

**Note:**
This blog is intended to record what I learned in *Big Data Analysis with Scala and Spark*, rather than share the course materials. So there are two things that I won't put up: my solution to assignments and topics which I've mastered already.
PLEASE NOTIFY ME if I broke [Coursera Honor Code](https://learner.coursera.help/hc/en-us/articles/209818863-Coursera-Honor-Code) accidentally.

## Week 1

### Introduction, Logistics, What You'll Learn

- this course is about distributed data parallelism in Spark.
- the contents of this course:
    - extending data parallel paradigm to the distributed case, using Spark.
    - Spark’s programming model.
    - distributing computation, and cluster topology in Spark.
    - how to improve performance; data locality, how to avoid recomputation and shuffles in Spark.
    - relational operations with DataFrames and Datasets.
- recommended books and resouces:
    - Learning Spark (2015) (Holden Karau): covering the basics.
    - Spark in Action (2017) (Petar Zecevic): covering the basics, full of examples, first time learning.
    - High Performance Spark (Holden Karau): deeper, how Spark executes jobs, how to squeeze better performance.
    - Advanced Analytics with Spark (2015) (Sandy Ryza): mainly focusing on data science.
    - Mastering Apache Spark 2 (Jacek Laskowski).

### Data-Parallel to Distributed Data-Parallel

- shared memory data parallelism:
    - split the data.
    - workers/threads independently operate on the data shards in parallel.
    - combine when done (if necessary) .
- distributed data parallelism:
    - split the data over several nodes.
    - nodes independently operate on the data shards in parallel.
    - combine when done ( if necessary).
- distributed data parallalism has a new concern: network latency between workers.
- Spark implements a distributed data parallel model called Resilient Distributed Datasets (RDDs).

### Latency

- distribution introduces important concerns beyond what we had to worry about when dealing with parallelism in the shared memory case:
    - partial failure: crash failures of a subset of the machines involved in a distributed computation .
    - latency: certain operations have a much higher latency than other operations due to network communication.
- latency cannot be masked completely.
- fault-tolerance in Hadoop/MapReduce comes at a cost.
    - between each map and reduce step, in order to recover from potential failures, Hadoop/MapReduce shuffles its data and write intermediate data to disk.
- while spark achieves fault tolerance using ideas from functional programming.
    - keep all data immutable and in-memory.
    - all operations on data are just functional transformations, like regular Scala collections.
    - fault tolerance is achieved by replaying functional transformations over original dataset.

### RDDs, Spark's Distributed Collection

- RDDs seem a lot like **immutable** sequential or parallel Scala collections.
    - combinators on RDDs: `map`, `flatMap`, `filter`, `reduce`, `fold`, `aggregate`, ...
- difference between two signatures of `aggregate` methods:
```scala
aggregate[B](z: => B)(seqop: (B, A)=> B, combop: (B, B) => B): B // Scala, by-name
aggregate[B](z: B)(seqop: (B, A)=> B, combop: (B, B) => B): B // Spark RDD, by-value
/**
  * I guess aggregate of Spark RDD does not use by-name evaluation because
  * the computation actually occurs on every worker node, which means the
  * initial value (by-name parameter) would be evaluated same times as
  * the number of worker nodes. It does cause performance loss, I guess.
  */
```
- RDDs can be created in two ways:
    - transforming an existing RDD by high-order functions.
    - from a `SparkContext` (or `SparkSession`) object, which defines a handful of methods which can be used to create and populate a new RDD:
    ```scala
    parallelize // convert a local Scala collection to an RDD .
    textFile // read a text file from HDFS or a local file system and return an RDD of String
    ```

### RDDs: Transformation and Actions

- for Scala collection:
    - transformers: return new collections as results. `map`, `filter`, `flatMap`, `groupBy`.
    - accessors: return singles values. `reduce`, `fold`, `aggregate`.
- similarly, Spark defines transformations and actions on RDDs:
    - transformations: return new RDDs. It's `lazy` that its result is not immediately computed.
    - actions: compute a result based on an RDD, and either returned or saved to an external storage system. It's `eager` (instead of `lazy`).
- laziness/eagerness is how we can limit network communication using the programming model.
- common transformations (lazy):
    - `map`: `map[B](f: A => B): RDD[B]`.
    - `flatMap`: `flatMap[B](f: A => TraversableOnce[B])`.
    - `filter`: `filter(pred: A => Boolean)`.
    - `distinct`: `distinct(): RDD[B]`.
- common actions (eager):
    - `collect`: `collect(): Array[T]`, return all elements from RDD.
    - `count`: `count(): Long`, return the number of elements in the RDD.
    - `take`: `take(n: Int): Array[T]`, return the first `n` elements of the RDD.
    - `reduce`: `reduce(op: (A, A) => A)`.
    - `foreach`: `foreach(f: T => Unit)`.
- Spark computes RDDs the first time they are used in an action, which helps when processing large amounts of data.
- Spark leverages this by analyzing and optimizing the chain of operations before executing it.
```scala
val firstlogsWithErrors = lastYearslogs.filter(_.contains("ERROR")) .take(10)
/**
  * The execution of filter is deferred until the take action is applied.
  * As soon as 10 elements of the filtered RDD have been computed,
  * firstLogsWithErrors is done.
  */
```
- transformations on two RDDs:
    - `union`: `union(other: RDD[T]): RDD[T]`.
    - `intersection`: `intersection(other: RDD[T]): RDD[T]`.
    - `subtract`: `subtract(other: RDD[T]): RDD[T]`.
    - `cartesian`: `cartesian[U](other: RDD[U]): RDD[(T, U)]`.
- other useful actions
    - `takeSample`: `takeSample(withRepl: Boolean, num: Int): Array[T]`, return an array of a random sample of `num` elements of the dataset, with or without replacement.
    - `takeOrdered`: `takeOrdered(num: Int)(implicit ord: Ordering[T]): Array[T]`, return the first `n` elements of the RDDs using either their natural order or a custom comparator.
    - `saveAsTextFile`: `saveAsTextFile(path: String): Unit`, write the elements of the dataset as a text file in the local filesystem or HDFS.
    - `saveAsSequenceFile`: `saveAsSequenceFile(path: String): Unit`, write the elements of the dataset as a Hadoop SequenceFile in the local filesystem or HDFS.

### Evaluation in Spark: Unlike Scala Collections!

- by default, RDDs are re-computed every time you run an action on them, which can be expensive if the dataset is used more than once.
- luckily, Spark allows us to control what is cached in memory.
    - to tell Spark to cache an RDD in memory, simply call `persist()` or `cache()` on it.
    ```scala
    val lastYearsLogs: RDD[String] = ...
    // logsWithErrors is cached in memory.
    val logsWithErrors = lastYearsLogs.filter(_.contains("error")).persist()
    val firstLogsWithErrors = logsWithErrors.take(10)
    val numErrors = logsWithErrors.count() // faster than no-caching
    ```
- there are many ways to configure how your data is persisted.
    - in memory as regular Java objects.
    - on disk as regular Java objects.
    - in memory as serialized Java objects (more compact).
    - on disk as serialized regular Java objects (more compact).
    - both in memory and disk (spills over to disk to avoid re-computation).
- one of the most common performance bottlenecks of newcomers to Spark arises from unknownly re-evaluating several transformations when caching coud be used.

### Cluster Topology Matters!

- a Spark application is a set of processes running on a cluster.
- all these processes are coordinated by the `driver program`.
- the driver is:
    - the process where the main() method of your program runs.
    - the process running the code that creates a `SparkContext`, creates RDDs, and stages up or sends off transformations and actions.
- these processes that run computations and store data for your application are `executor`s.
- executors:
    - run the tasks that represent the application.
    - return computed results to the driver.
    - provide in-memory storage for cached RDDs.
- execution of a Spark program:
    - the driver program runs the Spark application, which creates a SparkContext upon start-up.
    - the SparkContext connects to a cluster manager (e.g., Mesos/YARN) which allocates resources.
    - spark acquires executors on nodes in the cluster, which are processes that run computations and store data for your application.
    - next, driver program sends your application code to the executors.
    - finally, SparkContext sends tasks for the executors to run.
- programmers should know where their code is running to avoid unexpected output or side-effects.

## Week 2

### Reduction Operations

- reduction operations: walk through a collection and combine neighboring elements of the collection together to produce a single combined result.
- properties of `aggregate`:
    - parallelizable. (like 'fold')
    - possible to change the return type. (like 'foldLeft')
- Spark does not support serial `foldLeft` or `foldRight` because doing things serially across a cluster requires lots of synchronization, which is actually difficult.
- as you will realize after experimenting with Spark a bit, much of the time when working with large-scale data, your goal is to **project down from larger/more complex data types**.

### Pair RDDs

- most common in world of big data processing: operating on data in the form of key-value pairs.
- in Spark, distributed key-value pairs are `PairRDD`.

### Transformations and Actions on Pair RDDs

- transformations:
    - `groupByKey`: `def groupByKey(): RDD[(K, Iterable[V])]`.
    - `reduceByKey`: `def reduceByKey(func: (V, V) => V): RDD[(K, V)]`, can be thought of as a combination of `groupByKey` and `reduce`-ing on all the values per key. It's more efficient though, than using each separately.
    - `mapValues`: ` def mapValues[U] (f: V => U) : RDD [(K, U)]`.
    - `keys`: `def keys: RDD[K]`, return an RDD with the keys of each tuple.
    - `join`.
    - `leftOuterJoin`/`rightOuterJoin`.
- actions:
    - `countByKey`: `def countByKey(): Map[K, Long]`.

### Joins

- two kinds of joins:
    - inner joins (`join`).
    - outer joins (`leftOuterJoin`, `rightOuterJoin`).
- the key difference between the two is what happens to the keys when both RDDs don't contain the same key.
- inner joins return a new RDD containing combined pairs whose keys are present in both input RDDs.
    - `def join[W](other: RDD[(K, W)]): RDD[(K, (V, W))]`.
- outer joins return a new RDD containing combined pairs whose keys don't have to be present in both input RDDs.
    - `def leftOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (V, Option[W]))]`
    - `def rightOuterJoin[W](other: RDD[(K, W)]): RDD[(K, (Option[V], W))]`

## Week 3

### Shuffling: What it is and why it's important

- shuffling: move data from one node to another to be grouped with its key.
    - shuffles can be an enormous hit due to network latency.
- `groupByKey` results in one key-value pair per key. And this single key-value pair cannot span across multiple worker nodes.
- conceptually, `reduceByKey` can be thought of as a combination of first doing `groupByKey` and then `reduce`-ing on all the values grouped per key.
    - but it's more efficient though, than using each separately.
    - it is because an pre-`reduceByKey` can be done in each executor to reduce the amount of data sent over the network.
    - this can result in non-trival gains in performance!

### Partitioning

- partitioning: how Spark know which key to put on which machine. The data within an RDD is split into several partitions.
- properties of partitions:
    - partitions never span multiple machines. i.e., tuples in the same partitions are guaranteed to be on the same machine.
    - each machine in the cluster contains one or more partitions.
    - the number of partitions to use is configurable. By default, it equals to the total number of cores on all executor nodes.
- two kinds of partitioning available in Spark:
    - hash partitioning: attempts to spread data evenly across partitions based on the key.
    - range partitioning: may be more efficient when Pair RDDs contain keys that have an ordering defined (`Int`, `String`).
- customizing a partitioning is only possible on Pair RDDs.
- using a range partitioner, keys are partitioned according to:
    - an ordering of keys.
    - a set of sorted ranges of keys.
- there are two ways to create RDDs with specific partitionings:
    - call `partitionBy` on an RDD, providing an explicit Partitioner.
    - using transformations that return RDDs with specific partitioners.

**call `partitionBy` on an RDD**:

``` scala
val pairs = purchasesRdd.map(p => (p.customerld, p.price))
val tunedPartitioner = new RangePartitioner(8, pairs)
val partitioned = pairs.partitionBy(tunedPartitioner).persist()
```
- creating a `RangePartitioner` requires:
    - specifying the desired number of partitions.
    - providing a Pair RDD with ordered keys. This RDD is sampled to create a suitable set of sorted ranges.
- **NOTE**: the result of `partitionBy` should always be persisted. Otherwise, the partitioning is repeatedly applyed (involving shuffling) each time the partitioned RDD is used.

** partitioning data using transformations**:
- partitioner from parent RDD:
    - Pair RDDs that are the result of a transformation on a partitioned Pair RDD typically is configured to use the hash partitioner that was used to construct it.
- automatically-set partitioners:
    - some operations on RDDs automatically result in an RDD with a known partitioner - for when it makes sense.
    - by default, when using `sortByKey`, a RangePartitioner is used. Further, the default partitioner when using `groupByKey`, is a HashPartitioner.
- operations on Pair RDDs that hold to (and propagate) a partitioner:
    - `cogroup`.
    - `groupWith`.
    - `join`.
    - `leftOuterJoin`.
    - `rightOuterJoin`.
    - `groupByKey`.
    - `reduceByKey`.
    - `foldByKey`.
    - `combineByKey`.
    - `partitionBy`.
    - `sort`.
    - `mapValues` (if parent has a partitioner).
    - `flatMapValues` (if parent has a partitioner).
    - `filter` (if parent has a partitioner).
- all other operations will produce a result without a partitioner.
- why `map` or `flatMap` do not preserve partitioner in their result RDDs?
    - because it's possible for `map` or `flatMap` to change the key.
    - the previous partitioner would no longer make sense when key is changed.
    - hence `mapValues` enables us to still do `map` transformations without changing the keys, thereby preserving the partitioner.

### Optimizing with Partitioners

- using range partitioners we can optimize our earlier use of `reduceByKey` so that it does not involve any shuffling over the network at all!
- grouping (`groupByKey`) is done using a hash partitioner with default parameters.
- rule of thumb: a shuffle can occur when the resulting RDD depends on other elements from the same RDD or another RDD.
- sometimes one can be clever and avoid much or all network communication while still using an operation like join via smart partitioning
- we can figure out whether a shuffle has been planned/executed via:
    - the return type of certain transformations (`ShuffledRDD`).
    - using function `toDebugString` to see its execution plan.
    ```scala
    partitioned.reduceByKey((v1, v2) => (v1 ._1 + v2._1, v1 ._2 + v2._2))
            .toDebugString
    ```
- operations that might cause a shuffle:
    - `cogroup`.
    - `groupWith`.
    - `join`.
    - `leftOuterJoin`.
    - `rightOuterJoin`.
    - `groupByKey`.
    - `reduceByKey`.
    - `combineByKey`.
    - `distinct`.
    - `intersection`.
    - `repartition`.
    - `coalesce`.
- there are a few ways to use operations that might cause a shuffle and to still avoid much or all network shuffling.
    - `reduceByKey` running on a pre-partitioned ROD will cause the values to be computed locally, requiring only the final reduced value has to be sent from the worker to the driver.
    - `join` called on two RDDs that are pre-partitioned with the same partitioner and cached on the same machine will cause the `join` to be computed locally, with no shuffling across the network.

### Wide vs Narrow Dependencies

- computations on RDDs are represented as a lineage graph: a Directed Acyclic Graph (DAG) representing the computations done on the RDD.
- RDDs are made up of 2 important parts (but are made up of 4 parts in total):
    - `Partitions`: atomic pieces of the dataset; one or many per compute node.
    - `Dependencies`. models relationship between this RDD and its partitions with the RDD(s) it was derived from.
    - `A function` for computing the dataset based on its parent RDDs.
    - `Metadata` about its partitioning scheme and data placement.
- in fact, RDD dependencies encode when data must move across the network.
- transformations can have two kinds of dependencies:
    - narrow dependencies: each partition of the parent RDD is used by at most one partition of the child RDD.
    - wide dependencies: each partition of the parent RDD may be depended on by multiple child partitions.
- narrow dependencies mean the transformation can be:
    - fast.
    - no shuffle necessary.
    - optimizations like pipelining possible.
- while wide dependencies are:
    - slow.
    - require all or some data to be shuffled over the network.
- transformations with narrow dependencies:
    - `map`.
    - `mapValues`.
    - `flatMap`.
    - `filter`.
    - `mapPartitions`.
    - `mapPartitionsWithIndex`.
- transformations with wide dependencies:
    - `cogroup`.
    - `groupWith`.
    - `join`.
    - `leftOuterJoin`.
    - `rightOuterJoin`.
    - `groupByKey`.
    - `reduceByKey`.
    - `combineByKey`.
    - `distinct`.
    - `intersection`.
    - `repartition`.
    - `coalesce`.
- we could use `dependencies` method on RDD to return a sequence of denpendency objects, which are actually the dependencies used by Spark's scheduler to know how this RDD depends on other RDDs.
``` scala
val pairs = wordsRdd.map(c => (c, 1))
        .groupByKey()
        .dependencies
```
- narrow dependency objects:
    - `OneToOneDependency`.
    - `PruneDependency`.
    - `RangeDependency`.
- wide dependency objects:
    - `ShuffleDependency`.
- `toDebugString` prints out a visualization of the RDD's lineage, and other information pertinent to scheduling.
- lineage graphs are the key to fault tolerance in Spark.
    - along with keeping track of dependency information between partitions as well, this allows us to:
    - recover from failures by recomputing lost partitions from lineage graphs.
- recomputing missing partitions fast for narrow dependencies, but slow for wide dependencies.


