---
title: 'Difference Between Val, Lazy Val and Def'
date: 2018-04-09 21:38:16
tags: [Scala]
---

1. `val`
   ```scala
   scala> val a = { println("hello, world."); 3 }
   hello, world.
   a: Int = 3

   scala> a
   res0: Int = 3
   ```

2. `lazy val`
   ```scala
   scala> lazy val a = { println("hello, world."); 3 }
   a: Int = <lazy>

   scala> a
   hello, world.
   res1: Int = 3

   scala> a
   res2: Int = 3
   ```

3. `def`
   ```scala
   scala> def a = { println("hello, world."); 3 }
   a: Int

   scala> a
   hello, world.
   res4: Int = 3

   scala> a
   hello, world.
   res5: Int = 3
   ```



---

*TBD*

