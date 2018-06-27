---
title: '[Coursera] Functional Programming Principles in Scala 课程笔记'
date: 2018-06-22 16:33:34
tags: [Study Notes, Scala, Functional Programming]
categories:
- [Open Course, Coursera]
---

**Note:**
This blog is intended to record what I learned in *Functional Programming Principles in Scala*, rather than share the course materials. So there are two things that I won't put up: my solution to assignments and topics which I've mastered already.
PLEASE NOTIFY ME if I obeyed [Coursera Honor Code](https://learner.coursera.help/hc/en-us/articles/209818863-Coursera-Honor-Code) accidentally.

## Week 1

### Lecture 1.1 - Programming Paradigms

- three main programming paradigms: `imperative programming`, `functional programming` and `logic programming`. (`object-oriented programming` is orthogonal to them)
- the largest problem of `imperative programming` is not suitable for scaling up (limited by the Von Neuman bottleneck).
- `FP` = focusing on functions (wider sence, Scala); programming without mutable variables, assignments and imperative control structures (restricted sence, Pure Lisp).
- functions are first-class citizens in a FP language, which means you can do with a function that you could do with any other piece of data.
- recommended books: SICP (more about functional programming), Programming in Scala (more about Scala), Scala for the Impatient (for people with a Java background), Scala in Depth (a bit further than others).
- benefits of FP: simpler reasoning princples, better modularity, good for exploiting parallelism for multicore and cloud computing.

### Lecture 1.2 - Elements of Programming

- evaluation of function applications: 1. evaluate all arguments, from left to right; 2. replace function by its right-hand side, and at the same time; 3. replace the formal arguments by the actual arguments.
- substitution model: scheme of expression evaluation (formaized in the $\lambda-calculas$, which is the foundation of FP).
- call-by-value and call-by-name:
  - both strategies reduce to the same final value as long as: the reduced expression consists of pure functions and both evaluations terminate.
  - call-by-value: reduces arguments to values before rewriting the function application; evaluate every argument only once.
  - call-by-name: apply the function to unreduced arguments; an argument won't be evaluated when it's unused in the evaluation of the function body.

### Lecture 1.3 - Evaluation Strategies and Temination

- if call-by-value evaluation of an expression terminates, then call-by-name terminates too.
- the other direction is not true
- Scala uses call-by-value by default; But if the type of a function parameter starts with `=>`, it uses call-by-name.
``` scala
def foo(x: Int, y: => Int): Int = x
```

### Lecture 1.4 - Conditionals and Value Definitions

- the `def` form is `by-name` (evaluated on each use); `val` is `by-value`.

### Lecture 1.5 - Example: Square roots with Newton's method

### Lecture 1.6 - Blocks and Lexical Scope

- if there are more than one statements on a line, they need to be separated by semicolons.
- how to write expressions that span several lines:
    - write the multi-line expression in parentheses, because semicolons are never inserted inside (...).
    - write the operator on the first line, because this tells the Scala compiler that the expression is not yet finished.
    ```scala
    a +
    b
    ```

### Lecture 1.7 - Tail Recursion

- tail recursion: if a function calls itself as its last action, the function’s stack frame can be reused.
- we can require a function is tail-recursive using a `@tailrec` annotation.

## Week 2

### Lecture 2.1 - High-Order Functions

- functional languages treat functions as first-class values.
- higher order functions: functions that take other functions as parameters or return other functions as results.
- anonymous functions: function literals.

### Lecture 2.2 - Currying - Principles of Functional Programming

- the type of `sum` is `(Int => Int) => (Int, Int) => Int`:
``` scala
def sum(f: Int => Int)(x: Int, y: Int): Int = ...
```
- function types associate to the right, so `Int => Int => Int` is equivalent to `Int => (Int => Int)`.

### Lecture 2.3 - Example: Finding Fixed Points

### Lecture 2.4 - Scala Syntax Summary

```
Type            = SimpleType | FunctionType
FunctionType    = SimpleType ‘= > ’ Type
                | ‘( ’ [ Types ] ‘) ’ ‘= > ’ Type
SimpleType      = Ident
Types           = Type { ‘ , ’ Type }

Expr            = InfixExpr | FunctionExpr
                | if ‘( ’ Expr ‘) ’ Expr else Expr
InfixExpr       = PrefixExpr | InfixExpr Operator InfixExpr
Operator        = ident
PrefixExpr      = [ ‘+ ’ | ‘-’ | ‘! ’ | ‘~ ’ ] SimpleExpr
SimpleExpr      = ident | literal | SimpleExpr ‘. ’ ident
                | Block
FunctionExpr    = Bindings ‘= > ‘ Expr
Bindings        = ident [ ‘: ’ SimpleType ]
                | ‘( ’ [ Binding { ‘ , ’ Binding }] ‘) ’
Binding         = ident [ ‘: ’ Type ]
Block           = ‘{ ’ { Def ‘; ’} Expr ‘} ’

Def             = FunDef | ValDef
FunDef          = def ident { ‘( ’ [ Parameters ] ‘) ’}
                    [ ‘: ’ Type ] ‘= ’ Expr
ValDef          = val ident [ ‘: ’ Type ] ‘= ’ Expr
Parameter       = ident ‘: ’ [ ‘= > ’ ] Type
Parameters      = Parameter { ‘ , ’ Parameter }
```

### Lecture 2.5 - Functions and Data

### Lecture 2.6 - More Fun With Rationals

- data abstraction: the ability to choose different implementation of the data without affecting clients.
- `require` and `assert`: require is used to enforce a precondition on the caller of a function, while assert is used as to check the code of the function itself.

### Lecture 2.7 - Evaluation and Operators

- the precedence of an operator is determined by its first character, and the following table lists the characters in increasing order of priority precedence:
```
(all letters)
|
^
&
< >
= !
:
+ -
* / %
(all other special characters)
```
- exercise:
``` scala
a + b ^? c ?^ d less a ==> b | c
// is equivalent to
((a + b) ^? (c ?^ d)) less ((a ==> b) | c)
```

## Week 3

### Lecture 3.1 - Class Hierarchies

- abstract classes can contain members which are missing an implementation.
- no instances of an abstract class can be created with `new`.
- dynamic dispatch of methods (in Object-Oriented Language) is analogous to calls to higher-order functions (in Functional Languages):
    - because the code thats gets executed on functional method call is not known statically, but it's determined by the runtime value that is passed.

### Lecture 3.2 - How Classes Are Organized

- a class can only have one superclass, but it could extends multiple `trait`s.
- `trait` can contains fields and concrete methods, but it cannot have parameters.
- `Null` is a subtype of every class that inherits from `Object`; it is incompatible with subtypes of `AnyVal`.

### Lecture 3.3 - Polymorphism

- type parameters do not affect evaluation in Scala.
    - type erasure: we can assume that all type parameters and type arguments are removed before evaluating the program.
    - languages that use type erasure: Scala, Java, Haskell, ML, Ocaml.
    - languages that keep the type parameters around run time: C++, C#, F#.
- polymorphsim has two principal forms: subtyping and generics:
    - subtyping: instances of a subclass can be passed to a base class.
    - generics: instances of a class or function are created by type parameterization.

## Week 4

### Lecture 4.1 - Objects Everywhere

- a pure object oriented language is one which every value is an object.
- define `Boolean` as a class from first principles:
``` scala
package idealized.scala
abstract class Boolean {
    def ifThenElse[T](t: => T, e: => T): T
    def && (x: => Boolean): Boolean = ifThenElse(x, false)
    def || (x: => Boolean): Boolean = ifThenElse(true, x)
    def unary_!: Boolean = ifThenElse(false, true)
    def == (x: Boolean): Boolean = ifThenElse(x, x.unary_!)
    def != (x: Boolean): Boolean = ifThenElse(x.unary_!, x)
}

object true extends Boolean {
    def ifThenElse[T](t: => T, e: => T) = t
}
object false extends Boolean {
    def ifThenElse[T](t: => T, e: => T) = e
}
```

### Lecture 4.2 - Functions as Objects

- function values are treated as objects in Scala:
    - the function type `A => B` is just an abbreviation for class `scala.Function1[A, B]`.

### Lecture 4.3 - Subtyping and Generics

- `A <: B` is an upper bound of the type parameter `A` which means `A` is a subtype of `B`.
- `A >: B` is an lower bound of the type parameter `A` which means `A` is a supertype of `B`.
- mixed bound is possible: `A >: B <: C` would restrict `A` any type on the interval between `B` and `C`.
- `Liskov Substitution Principle`: let q(x) be a property provable about objects x of type B, then q(y) should be provable for objects y of type A where A <: B

### Lecture 4.4 - Variance (Optional)

- (roughly speaking) a type that accepts mutations of its elements should not be covariant, but immutable types can be covariant if some conditions are met.
- covariant and contravariant:
    - given `A <: B`, if `C[A] <: C[B]`, `C` is covariant; if `C[A] >: C[B]`, `C` is covariant; if neither `C[A]` nor `C[B]` is a subtype of the other, `C` is nonvariant.
    ``` scala
    class C[+A] { ... } // C is covariant
    class C[-A] { ... } // C is contravariant
    class C[A] { ... } // C is nonvariant
    ```
- functions must be contravariant in their argument types and covariant in their result types:
    - for a function, if `A2 <: A1` and `B1 <: B2`, then `A1 => B1 <: A2 => B2`.
    - covariant type parameters can only appear in method results.
    - contravariant type parameters can only appear in method parameters.
    - invariant type parameters can appear anywhere.

- sometimes we have to put in a bit of work to make a class covariant.
``` scala
trait List[+T] {
    // the following code does not type-check because T is covariant.
    // List[IntSet].prepend(Empty) works but List[NonEmpty].prepend(Empty) does not work.
    def prepend(elem: T): List[T] = new Cons(elem, this)

    // we can use a lower bound to solve this problem.
    def prepend [U >: T] (elem: U): List[U] = new Cons(elem, this)
}
```

### Lecture 4.5 - Decomposition

### Lecture 4.6 - Pattern Matching

- pattern matching is a generalization of switch from C/Java to class hierarchies.
    - a constructor pattern `C(p1, ..., pn)` matches all the values of type `C` (or a subtype) that have been constructed with arguments matching the patterns `p1, ..., pn`.
    - a variable pattern `x` matches any value, and binds the name of the variable to this value.
    - a constant pattern `c` matches values that are equal to `c` (in the sense of `==`)

### Lecture 4.7 - Lists

- lists are immutable; lists are recursive, while arrays are flat.
- like arrays, lists are homogeneous that all elements of a list must all have the same type.
- operators ending in `:` associate to the right.

## Week 5

### Lecture 5.1 - More Functions on Lists

### Lecture 5.2 - Pairs and Tuples

### Lecture 5.3 - Implicit Parameters

- `scala.math.Ordering[T]` provides ways to compare elements of type `T`.
- if a function takes an implicit parameter of type `T`, the compiler will search an implicit definition that:
    - is marked `implicit`.
    - has a type compatible with `T`.
    - is visible at the point of the function call, or is defined in a companion object associated with T.
- if there is a single (most specific) definition, it will be taken as actual argument for the implicit parameter; otherwise it's an error.

### Lecture 5.4 - High-Order List Functions

### Lecture 5.5 - Reduction of Lists

- `reduceLeft` inserts a given binary operator between adjacent elements of a list.
- `foldLeft` is like `reduceLeft` but takes an accumalator.
- `foldLeft` and `reduceLeft` produces trees which lean to the left, while `foldRight` and `reduceRight` produces trees which lean to the right.

### Lecture 5.6 - Reasoning About Concat

### Lecture 5.7 - A Larger Equational Proof on Lists

## Week 6

### Lecture 6.1 - Other Collections

- `Vector` has more evenly balanced access patterns than `List` based on its structure design.
- a common base class of `List` and `Vector` is `Seq`, the class of all sequences, which is a subclass of `Iterable`.
``` Scala
Seq, Set, Map <: Iterable
Vector, List, Range <: Seq
// Array and String support the same operations as Seq and can implicitly be converted to sequences where needed,
// but they cannot be subclasses of Seq because they come from Java.
```
- `Range` represents a sequence of evenly spaced integers, which has three operators: `to` (inclusive), `until` (exclusive) and `by` (step value).
``` scala
1 to 4 // 1, 2, 3, 4
1 until 4 // 1, 2, 3
1 to 4 by 2 // 1, 3
```

### Lecture 6.2 - Combinatorial Search and For-Expressions

- a `for`-expression is of the form: `for (s) yield e` where:
    - `s` is a sequence of generators or filters,
    - `e` is an expression whose value is returned by an iteration.
    - a generator is of the form `p <- e`.
    - a filter is of the form `if f`.
    - the sequence must start with a generator.
    - if there're multiple generators in the sequence, the last generators vary faster than the first one.

### Lecture 6.3 - Combinatorial Search Example

- `Set` is different with `Seq`:
    - `Set` is unordered.
    - `Set` does not have duplicate elements.
    - the fundamental operation on `Set` is `contains`.

### Lecture 6.4 - Maps

- repeated parameter: `T*` means you can pass variable number of parameter with type `T`.

### Lecture 6.5 - Putting the Pieces Together
