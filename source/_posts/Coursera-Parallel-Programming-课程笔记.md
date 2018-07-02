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
```
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

### Scala Parallel Collections

### Splitters and Combiners
