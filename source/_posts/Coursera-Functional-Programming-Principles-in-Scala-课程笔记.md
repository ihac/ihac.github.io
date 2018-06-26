---
title: '[Coursera] Functional Programming Principles in Scala 课程笔记'
date: 2018-06-22 16:33:34
tags: [Study Notes, Scala, Functional Programming]
categories:
- [Open Course, Coursera]
---

**Note:**
This blog is intended to record what I learned in *Functional Programming Principles in Scala*, rather than sharing the course materials. So there are two things that I won't put up: my solution to assignments and topics which I've mastered already.

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
## Week 4
## Week 5
## Week 6
