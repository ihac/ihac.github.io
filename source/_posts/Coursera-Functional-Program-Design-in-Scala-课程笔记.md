---
title: '[Coursera] Functional Program Design in Scala 课程笔记'
date: 2018-06-26 21:30:53
tags: [Study Notes, Scala, Functional Programming]
categories:
- [Open Course, Coursera]
---

**Note:**
This blog is intended to record what I learned in *Functional Program Design in Scala*, rather than sharing the course materials. So there are two things that I won't put up: my solution to assignments and topics which I've mastered already.

## Week 1

### Recap: Functions and Pattern Matching

### Recap: Collections

### Lecture 1.1 - Queries with For

- the for notation is essentially equivalent to the common operations of query languages for databases.

### Lecture 1.2 - Translation of For

- the Scala compiler translates for-expressions in terms of map, flatMap and a lazy variant of filter.
``` scala
for (x <- e1) yield e2
// is translated to
e1.map(x => e2)

for (x <- e1 if f; s) yield e2
// is translated to
for (x <- e1.withFilter(x => f); s) yield e2
// withFilter is a variant of filter which does not produce an intermediate list

for (x <- e1; y <- e2; s) yield e3
// is translated to
x1.flatMap(x => for (y <- e2, s) yield e3)
```
- we can use `for` syntax for our own types as long as we define `map`, `flatMap` and `withFilter` for these types.

### Lecture 1.3 - Functional Random Generators

- we can use `self =>` at the begining of `class`/`trait` body to declare an alias for `this`.
``` scala
trait Generator[+T] {
    self => // an alias for ”this”.
    def generate: T
    def map[S](f: T => S): Generator[S] = new Generator[S] {
        def generate = f(self.generate)
    }
}
```

## Week 2
## Week 3
## Week 4
