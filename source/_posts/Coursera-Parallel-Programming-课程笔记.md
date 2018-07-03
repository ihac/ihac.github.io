---
title: '[Coursera] Parallel Programming 课程笔记'
date: 2018-06-28 21:22:24
tags: [Study Notes, Scala, Functional Programming]
categories:
- [Open Course, Coursera]
---

**Note:**
This blog is intended to record what I learned in *Parallel Programming*, rather than share the course materials. So there are two things that I won't put up: my solution to assignments and topics which I've mastered already.
PLEASE NOTIFY ME if I broke [Coursera Honor Code](https://learner.coursera.help/hc/en-us/articles/209818863-Coursera-Honor-Code) accidentally.

## Week 1

### Introduction to Parallel Computing

- at the beginning of 21 century processor frequency scaling hit the *power wall*.
- parallism and concurrency are closely related concepts:
    - parallel program uses parallel hardware to execute computation more quickly. Efficiency is its main concern (or speedup).
    - concurrent program may or may not execute multiple executions at the same time. It concerns about improving modularity, responsiveness and maintainability (or convenience).
- parallelism granularity:
    - bit-level parallelism: processing multiple bits of data in parallel.
    - instruction-level parallelism: executing different instructions from the same instruction stream in parallel.
    - task-level parallelism: executing separate instruction streams in parrel.
- different forms of parallel hardware.
    - multi-core processor.
    - symmetric multiprocessors.
    - graphics processing unit.
    - field-programmable gate arrays.
    - computer clusters.

### Parallelism on the JVM I

### Parallelism on the JVM II

- memory model is a set of rules that describe how threads interact when accessing shared memory.
- Java Memory Model - the memory model for JVM:
    - two threads writing to separate locations in memory do not need synchroization.
    - a thread `X` that calls `join` on another thread `Y` is guaranteed to observe all the writes by thread `Y` after `join` returns.

### Running Computations in Parallel

### How Fast are Parallel Programs?

- how to estimate the performance of parallelism:
    - empirical measurement.
    - asymptotic analysis, which is important to understand how algorithms scale when input get larger or we have more hardware parallelism available.

### Benchmarking Parallel Programs

- when a JVM program starts, it undergoes a period of *warmup*, after which it achieves its maximum performance.
    - first, the program is interpreted.
    - then, parts of the program are compiled into machine code.
    - later, the JVM may choose to apply additional dynamic optimizations.
    - eventually, the program reaches steady state.

## Week 2

### Parallel Sorting

- implement `merge sort` in parallel.

### Data Operations and Parallel Mapping

- operations on `List` are not good for parallel implementation, since spliting a list in half and combining them takes linear time.
- this section mainly concentrate on the parallel implementation of `Map` operation on `Array` and `Tree`.

### Parallel Fold (Reduce) Operation

- operation `f: (A, A) => A` is associative iff for every `x`, `y`, `z`: `f(x, f(y, z)) = f(f(x, y), z)`

### Associativity I

- operation `f: (A, A) => A` is commutative iff for every `x`, `y`: `f(x, y) = f(y, x)`
- floating point addition is commutative but not associative:
``` scala
scala> val e = 1e-200
e: Double = 1.0E-200
scala> val x = 1e200
x: Double = 1.0E200
scala> val mx = -x
mx: Double = -1.0E200

scala> (x + mx) + e
res2: Double = 1.0E-200
scala> x + (mx + e)
res3: Double = 0.0
scala> (x + mx) + e == x + (mx + e)
res4: Boolean = false
```

### Associativity II

- for `E(x,y,z) = f(f(x,y), z)`, we say arguments of E can rotate if E(x,y,z) = E(y,z,x), that is: `f(f(x,y), z) = f(f(y,z), x)`.
- if `f` is commutative and arguments of `E` can rotate then `f` is also associative.

### Parallel Scan (Prefix Sum) Operation

## Week 3

### Data-Parallel Programming

- task-parallel programming: a form of parallelization what distributes execution processes across computing nodes.
- data-parallel programming: a form of parallelization what distributes data across computing nodes.
- parallel for loop is the simplest form of data-parallel programming:
``` scala
for (i <- (0 until xs.length).par) {
    xs(i) = i
}
```
- goal of data-parallel scheduler: effiently balance the workload across processors without any knowledge about the `w(i) = #iterations`.

### Data-Parallel Operations I

- most collections in scala can become data-parallel by appending `.par`.
- operations `reduceLeft`, `reduceRight`, `scanLeft`, `scanRight`, `foldLeft`, `foldRight` must process the elements sequentially.
- the `fold` operation can process the elements in a reduction tree, so it can execute in parallel.

### Data-Parallel Operations II

- in order for the `fold` operation to work correctly, the following relations must hold:
``` scala
f(a, f(b, c)) == f(f(a, b), c) // associativity
f(z, a) == f(a, z) == a // neutral element
```
- we say that the neutral element z and the binary operator f must form a `monoid`.
- for `f = math.max`, we have `f(a, (b, c)) == f(f(a, b), c)` and `f(Int.MinValue, a) == f(a, IntMinValue) = a`:
```
def max(xs: Array[Int]): Int =
    xs.par.fold(Int.MinValue)(math.max)
```
- the `fold` operation can only produce values of the same type as the collection that it is called on.

### Scala Parallel Collections

- Scala collections hierarchy:
    - `Traversable[T]` – collection of elements with type `T`, with operations implemented using foreach.
    - `Iterable[T]` – collection of elements with type `T`, with operations implemented using iterator.
    - `Seq[T]` – an ordered sequence of elements with type `T`.
    - `Set[T]` – a set of elements with type `T` (no duplicates).
    - `Map[K, V]` – a map of keys with type `K` associated with values of type `V` (no duplicate keys).
- rule: never modify a parallel collection on which a data-parallel operation is in progress.
    - never write to a collection that is concurrently traversed.
    - never read from a collection that is concurrently modified

### Splitters and Combiners

- 4 abstractions for data-parallel collections: `iterators`, `splitters`, `builders`, `combiners`.
- `Iterator`:
``` scala
trait Iterator[A] {
    def next(): A
    def hasNext: Boolean
}
def iterator: Iterator[A] // on every collection
```
- the `Iterator` contract:
    - `next` can be called only if `hasNext` returns true.
    - after `hasNext` returns false, it will always return false.
- `Splitter`:
``` scala
trait Splitter[A] extends Iterator[A] {
    def split: Seq[Splitter[A]]
    def remaining: Int
}
def splitter: Splitter[A] // on every parallel collection
```
- the `Splitter` contract:
    - after calling `split`, the original splitter is left in an undefined state.
    - the resulting splitters traverse disjoint subsets of the original splitter.
    - `remaining` is an estimate on the number of remaining elements.
    - `split` is an efficient method – $O(log n)$ or better.
- `Builder`:
``` scala
trait Builder[A, Repr] {
    def +=(elem: A): Builder[A, Repr]
    def result: Repr
}
def newBuilder: Builder[A, Repr] // on every collection
```
- the `Builder` contract:
    - calling `result` returns a collection of type `Repr`, containing the elements that were previously added with `+=`.
    - calling `result` leaves the Builder in an undefined state.
- `Combiner`:
``` scala
trait Combiner[A, Repr] extends Builder[A, Repr] {
    def combine(that: Combiner[A, Repr]): Combiner[A, Repr]
}
def newCombiner: Combiner[T, Repr] // on every parallel collection
```
- the `Combiner` contract:
    - calling `combine` returns a new combiner that contains elements of input combiners.
    - calling `combine` leaves both original Combiners in an undefined state.
    - `combine` is an efficient method – $O(log n)$ or better.
