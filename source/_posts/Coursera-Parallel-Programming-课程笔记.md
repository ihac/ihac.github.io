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

### Data Operations and Parallel Mapping

### Parallel Fold (Reduce) Operation

### Associativity I

### Associativity II

### Parallel Scan (Prefix Sum) Operation
