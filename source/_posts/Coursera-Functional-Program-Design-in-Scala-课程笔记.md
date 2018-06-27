---
title: '[Coursera] Functional Program Design in Scala 课程笔记'
date: 2018-06-26 21:30:53
tags: [Study Notes, Scala, Functional Programming]
categories:
- [Open Course, Coursera]
---

**Note:**
This blog is intended to record what I learned in *Functional Program Design in Scala*, rather than share the course materials. So there are two things that I won't put up: my solution to assignments and topics which I've mastered already.
PLEASE NOTIFY ME if I obeyed [Coursera Honor Code](https://learner.coursera.help/hc/en-us/articles/209818863-Coursera-Honor-Code) accidentally.

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
    def map[S](f: T => S): Generator[S] = new Generator[S] {
        // we cannot use this here (but Generator.this is ok)
        def generate = f(self.generate)
    }
}
```
- define a trait `Generator[T]` that generates random values of type `T`:
``` scala
trait Generator[+T] {
    def generate: T
}

// some instances
val integers = new Generator[Int] {
    val rand = new java.util.Random
    def generate: Int = rand.nextInt()
}
val booleans = new Generator[Boolean] {
    def generate: Boolean = integers.generate > 0
}
val pairs = new Generator[(Int, Int)] {
    def generate = (integers.generate, integers.generate)
}
```
- how can we avoid the `new Generator` boilerplate since it's not trivial? Use `for` syntax:
``` scala
// Ideally, we would like to write:
val booleans = for (x <- integers) yield x > 0
def pairs[T, U](t: Generator[T], u: Generator[U]) = for {
    x <- t
    y <- u
} yield (x, y)

// they would be expanded to:
val booleans = integers.map(x => x > 0)
def pairs[T, U](t: Generator[T], u: Generator[U]) = t.flatMap(x => u.map(y => (x, y)))

// so we have to define map and flatMap for trait Generator:
trait Generator[+T] {
    self => // an alias for ”this”.
    def generate: T
    def map[S](f: T => S): Generator[S] = new Generator[S] {
        def generate = f(self.generate)
    }
    def flatMap[S](f: T => Generator[S]): Generator[S] = new Generator[S] {
        def generate = f(self.generate).generate
    }
}
```
- some helper functions:
``` scala
def single[T](x: T): Generator[T] = new Generator[T] {
    def generate = x
}
def choose(lo: Int, hi: Int): Generator[Int] =
    for (x <- integers) yield lo + x % (hi - lo)
def oneOf[T](xs: T*): Generator[T] =
    for (idx <- choose(0, xs.length)) yield xs(idx)
```
- [Exercise] how to implement a generator that creates random `Tree` objects?
``` scala
trait Tree
case class Inner(left: Tree, right: Tree) extends Tree
case class Leaf(x: Int) extends Tree

// my solution:
def trees: Generator[Tree] = for {
    isLeaf <- booleans
    tree <- if (isLeaf) leafs else inners
} yield tree
def leafs: Generator[Leaf] = for (x <- integers) yield Leaf(x)
def inners: Generator[Inner] = for {
    left <- trees
    right <- trees
} yield new Inner(left, right)
```
- random test: generate random test inputs.
``` scala
def test[T](g: Generator[T], runTimes: Int = 100)
    (f: T => Boolean): Unit = {
    for (i <- 0 until runTimes) {
        val t = g.generate
        assert(f(t), "assert failed for " + t)
    }
    println("passed %s tests".format(runTimes))
}
```

### Lecture 1.4 - Monads

- a `monad` is a parametric type `M[T]` with two operations: `flatMap` (in the literature, flatMap is more commonly called `bind`) and `unit`, that have to satisfy some laws:
``` scala
trait M[T] {
    def flatMap[U](f: T => M[U]): M[U]
}
def unit[T](x: T): M[T] ```
- `List` is a monad with `unit(x) = List(x)`; `Set` is monad with `unit(x) = Set(x)`; `Generator` is a monad with `unit(x) = single(x)`
- `map` can be defined for every monad as a combination of `flatMap` and `unit`:
``` scala
m map f = m flatMap (x => unit(f(x)))
```
- to qualify as a monad, a type has to satisfy three laws:
    - associativity:
    ``` scala
    m flatMap f flatMap g == m flatMap (x => f(x) flatMap g)
    ```
    - left unit:
    ``` scala
    unix(x) flatMap f == f(x)
    ```
    - right unit:
    ``` scala
    m flatMap unit == m
    ```
- [Note] you might find `List` doesn't obey the `left unit` rule since `List(1) flatMap (Set(_)) = List(1) != Set(1)`, this is because the monad law assumes `f: A => M[A]` (here `f: A => List[A]`). Refer to [link](https://stackoverflow.com/questions/45002864/monads-left-unit-law-does-not-seem-to-hold-for-lists-in-scala-are-scala-lists).
``` scala
List(1) flatMap (x => List(x, x)) = List(1, 1) == (x => List(x, x))(1)
```
- what is the significance of the laws with respect to `for` syntax?
    - associativity says essentially that one can “inline” nested for expressions:
    ``` scala
    for (y <- for (x <- m; y <- f(x)) yield y
        z <- g(y)) yield z
    ==
    for (x <- m;
        y <- f(x)
        z <- g(y)) yield z
    ```
    - right unit says:
    ``` scala
    for (x <- m) yield x == m
    ```
    - left unit  does not have an analogue for for-expressions.
- `Try` is not a monad since it breaks the `left unit` rule:
    - `Try(expr) flatMap f != f(expr)`: left-hand side will never raise a non-fatal exception whereas the right-hand side will raise any exception thrown by expr or `f`.
- `Try` trades one monad law for another law which is more useful in this context:
    - an expression composed from `Try`, `map`, `flatMap` will never throw a non-fatal exception.

## Week 2

### Lecture 2.1 - Structural Induction on Trees

### Lecture 2.2 - Streams

- streams is similar to lists, but their tail is evaluated only on demand.
``` scala
scala> Stream(1, 2, 3)
res0: scala.collection.immutable.Stream[Int] = Stream(1, ?)

scala> res0.tail
res1: scala.collection.immutable.Stream[Int] = Stream(2, ?)
```
- we use `#::` instead of `::` for stream prepending.
- how to use stream to improve efficiency?
``` scala
// it is inefficient because it constructs all prime numbers between 1000
// and 10000 in a list, while what we need is only the second one.
((1000 to 10000) filter isPrime)(1)

// using stream only needs evaluate the first two prime numbers.
((1000 to 10000).toStream filter isPrime)(1)
```

### Lecture 2.3 - Lazy Evaluation

- in a purely functional language an expression produces the same result each time it is evaluated.
- `lazy evaluation` means evaluting on first demand, storing the result of the first evaluation and re-using the stored result instead of recomputing.
    - it's not `by-name evaluation` where everything is recomputed.
    - it's not `restricted evaluation` for normal parameters and `val` definitions.
- Haskell use lazy evaluation by default, but Scala use stricted evaluation by default since it also supports mutable side effects (which might be inharmonious with lazy evaluation) in functions.

### Lecture 2.4 - Computing with Infinite Sequences

- define an infinity stream:
``` scala
def from(n: Int): Stream[Int] = n #:: from(n+1)
// all natural numbers:
val nats = from(0)
```
- calculate all prime numbers:
``` scala
def sieve(s: Stream[Int]): Stream[Int] =
    s.head #:: sieve(s.tail filter (_ % s.head != 0))
val primes = sieve(from(2))
```
- calculate the square root:
``` scala
def sqrtStream(x: Double): Stream[Double] = {
    def improve(guess: Double) = (guess + x / guess) / 2
    lazy val guesses: Stream[Double] = 1 #:: (guesses map improve)
    guesses
}
def isGoodEnough(guess: Double, x: Double) =
    math.abs((guess * guess - x) / x) < 0.0001
sqrtStream(4) filter (isGoodEnough(_, 4))
```

### Lecture 2.5 - Case Study: the Water Pouring Problem

## Week 3
## Week 4
