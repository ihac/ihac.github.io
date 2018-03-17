---
title: Scala学习笔记
date: 2017-07-27 14:43:05
tags: [Scala, Study Notes]
---

### Parameterize arrays with types
- If a method takes only one parameter, you can call it without a dot or parentheses.
``` scala
println(123) // equal to
Console.println(123) // equal to
Console println 123
```

- Scala is an object-oriented language in pure form: every value is an object and every operation is a method call.
``` scala
1 + 2 // actually invoking
(1).+(2)
```

- When you apply parentheses surrounding one or more values to a variable, Scala will transform the code into an invocation of a method named `apply` on that variable. It's a general rule.
``` scala
val testArray: Array[Int] = new Array[Int](5)
testArray(0) // get transformed into
testArray.apply(0)

val numNames = Array("zero", "one", "two")
// get transformed into
val numNames2 = Array.apply("zero", ...)
```

- Similarly, when an assignment is made to a variable to which parentheses and on e or more arguments have been applied, the compiler will transform that into an invocation of an update method that takes the arguments in parentheses as well as the object to the right of the equals sign.
```scala
testArray(0) = "hello" // get transformed into
testArray.update(0, "hello")
```

### Use lists
- List is immutable in Scala. It has a method named `:::` for list concatenation.
``` scala
List(1, 2) ::: List(3, 4) == List(1, 2, 3, 4)
```

- We can also use `::`(pronounced `cons`) to prepends a new element to the beginning of an existing list and returns the resulting list. Here `::` is a method of its right operand.(If a method is used in operator notation, such as `a * b`, the method is invoked on the left operand, as in `a.*(b)`——unless the method name ends in a colon)
``` scala
1 :: List(2, 3) == List(1, 2, 3)
// get transformed into
List(2, 3).::(1)

val oneTwoThree = 1 :: 2 :: 3 :: Nil
// here Nil is an empty list
```

- The time it takes to append to a list grows linearly with the size of the list, whereas prepending with `::` takes constant time. If you want to build a list efficiently by appending elements, you can prepend them and call reverse, or use ListBuilder.

### Use tuples

- The actual type of a tuple depends on the number of elements it contains and the types of those elements. Thus, the type of `(99, "Luftballons")` is `Tuple2[Int, String]`.

- The `_N` numbers for accessing the elements of a tuple are one-based, instead of zero-based, because starting with 1 is a tradition set by other languages with statically typed tuples.

### Use sets and maps

- The Scala API contains a base `trait` for sets, where a trait is similar to a Java interface. Scala then provides two subtraits, one for mutable sets and another for immutable sets.

- Method `->`, which you can invoke on any object in a Scala program, returns a two-element tuple containing the key and value.
``` scala
someMap += (1 -> "hello") // get transformed into
someMap += (1).->("hello")
someMap += ((1, "hello"))
```

### Classes, fields, and methods

- One important characteristic of method parameters in Scala is that they are vals, not vars. Never reassign a parameter inside a method in Scala.

- In the absence of any explicit return statement, a Scala method returns the last value computed by the method. The recommended style for methods is in fact to avoid having explicit, and especially multiple, return statements. Instead, think of each method as an expression that yields one value.

### Singleton objects

- When a singleton object shares the same name with a class, it is called that class's `companion object`. You must define both the class and its companion object in the same source file. A class and its companion object can access each other's private members.

- A singleton object that does not share the same name with a companion class is called a standalone object.

### Symbol literals

- `Symbol`s are interned in Scala, which means that if you write the same symbol literal twice, both expressions will refer to the exact same Symbol object.

### String interpolation

- In Scala, string interpolation is implemented by rewriting code at compile time.

### Any method can be an operator

- infix operator notation
``` scala
s.indexOf('a') == s indexOf 'a'
s.indexOf('a', 3) == s indexOf ('a', 3)
```
- prefix operator notation
``` scala
-2.0 == (2.0).unary_-
```

- postfix operator notation
```scala
s.toLowerCase == s toLowerCase
```

- In Scala, you can leave off empty parentheses on method calls. The convention is that you include parentheses if the method has side effects, such as `println()`, but you can leave them off if the method has no side effects, such as `toLowerCase` invoked on a `String`.

### Operator precedence and  tivity

- The associativity rule also plays a role when multiple operators of the same precedence appear side by side. If the methods end in `:` they are grouped right to left; otherwise, they are grouped left to right. For example, `a ::: b ::: c` is treated as `a ::: (b ::: c)`. But `a * b * c`, by contrast, is treated as `(a * b) * c`.

### Value of assignment

- In Java or C, assignments result in the value assigned, while in Scala, assignments always result in the unit value, `()`.
``` scala
while ((line = readline()) != "") { // wrong
  // do something
}
```
Here `line = readline()` will always be `()` and can never be `""`.


### Exception handling with try expressions

- `try-catch-finally` results in a value. The result is that of the try clause if no exception is thrown, or the relevant catch clause if an exception is throw and caught. If an exception is thrown but not caught, the expression has no result at all. The value computed in the finally clause, if there is one, is dropped.

- As in Java, if a finally clause includes an explicit return statement, or throws an exception, that return value or exception will "overrule" any previous one that originated in the try block or one of its catch clauses.
```scala
def f(): Int = try return 1 finally return 2 // f() == 2
def g(): Int = try 1 finally 2 // g() == 1
```

### Short forms of function literals

- This is called `target typing` because the targeted usage of an expression(in this case, an argument to someNumber.filter()) is allowed to influence the typing of that expression(in this case to determine the type of the x parameter).
```scala
val someNumbers = List(-11, -10, -5, 0, 5, 10)
someNumbers.filter((x: Int) => x > 0)
someNumbers.filter(x => x > 0)
```

### Placeholder syntax

- We can use underscores as placeholders for one or more parameters, so long as each parameter appears only one time within the function literal.
```scala
someNumbers.filter(_ > 0)
```

### Repeated parameters

- Scala allows you to indicate that the last parameter may be repeated as below. Thus, the type of args inside the echo function, which is declared as type `String*` is actually `Array[String]`. Nevertheless, an array cannot be passed as a repeated parameter.
```scala
def echo(args: String*) = args.map(println)
val arr = Array("hello", "world", "hac")
echo(arr) // compiler error
```

- To accomplish this, you'll need no append the array argument with a colon and an `_*` symbol as below. This notation tells the compiler to pass each element of array as its own argument to echo, rather than all of it as a single argument.
```scala
echo(arr: _*)
```

### Tail call optimization

- We can turn tail call optimization off by giving the following argument to the scala shell or to the scalac compiler.
``` scala
-g:notailcalls
```
- Scala only optimizes directly recursive calls back to the same function making the call. If the recursion is indirect, as in the following example of two mutually recursive functions, no optimization is possible:
``` scala
def isEven(x: Int): Boolean =
  if (x == 0) true else isOdd(x - 1)
def isOdd(x: Int): Boolean =
  if (x == 0) false else isEven(x - 1)
```
- You also won't get a tail-call optimization if the final call goes to a function value. Tail-call optimization is limited to situations where a method or nested function calls itself directly as its last operation, without going through a function value or some other intermediary.
```scala
val funValue = nextedFun _
def nestedFun(x: Int): Unit = {
  if (x != 0) { println(x); funValue(x - 1) }
}
```
### Evaluation strategies

- There are two evaluation strategies in Scala: `call by name` and `call by value`.
``` scala
def loop: Any = loop // never terminate
def foo(x: Any, y: Any) = x // call by value
def goo(x: Any, y: => Any) = x // call by name

foo(2, loop) // never terminate
goo(2, loop) = 2
```

- The `def` form is `call by name`, which means that its right hand side is evaluated on each use. However, the `val` form is `call by value`.

### Unary operator

- We override unary operator by using `unary_{operator}`.
``` scala
def unary_- : T = this.negative
```

### Class hierarchy

- There are 9 value classes in Scala: Byte, Short, Char, Boolean, Int, Long, Float, Double, Unit. You cannot create instances of these classes using `new`. Instead, all instances of these classes are written as literals. This is enforced by the "trick" that value classes are all defined to be both `abstract` and `final`.

### Trait

- You can do anything in `trait` definition that you can do in a class definition, and the syntax looks exactly the same, with only two exceptions.
	- A trait cannot have any "class" parameters.
	- In classes, `super call` is statically bound, while in traits, super calls are dynamically bound.

### Import

- In scala, an import selector can consist of the following:
	- A simple name `x`. This includes `x` in the set of imported names.
	- A renaming clause `x => y`. This makes the member named `x` visible under the name `y`.
	- A hiding clause `x => _`. This excludes `x` from the set of imported names.
	- A *catch-all* `_`. This imports all members except those members mentioned in a preceding clause. If a catch-all is given, it must come last in the list of import selectors.

### Access modifiers

- Java permits an outer class to access the private members of its inner classes, while Scala not.

- Access to protected members in Scala is also a bit more restrictive than in Java. In Scala, a protected member is only accessible from subclasses of the class in which the member is defined. While in Java, such accesses are also possible from other classes in the same package.

- Access modifiers can be augmented with qualifiers in Scala. A modifier form `private[X]` or `protect[X]` means that access is private or protect **up to** `X`, where `X` designates some enclosing package, class or singleton object.

---

*To Be Continued*
