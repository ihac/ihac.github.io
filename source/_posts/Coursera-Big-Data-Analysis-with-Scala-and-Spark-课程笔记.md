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
