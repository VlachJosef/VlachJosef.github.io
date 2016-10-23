---
title: Aux Pattern Evolution
date: 2016-10-23T09:51:20+01:00
layout: single
---

When learning [Shapeless](https://github.com/milessabin/shapeless) you will sooner or later encounter the so called _Aux Pattern_. There exists nice tutorial by Luigi Antonini  - [The Aux Pattern - A short introduction to the `Aux Pattern`](http://gigiigig.github.io/posts/2015/09/13/aux-pattern.html) which explains this pattern quite well and you will gain good understanding what problem this pattern is solving. I recommend you to go and read that blog post first and come back if you would feel that you are still missing something in a complete understanding of a said pattern. That was exactly my feeling and motivation to write this article.

In this article I'm going to take step back into the past and I'm going to show how `Aux Pattern` was initially implemented before the contemporary implementation made it into Shapeless 2.x.x.

As a driving example we will use following type class (in order to avoid any external dependecies):

<a name="unwrap-typeclass"></a>

```scala
trait Unwrap[T[_], R] {
  type Out
  def apply(tr: T[R]): Out
}
```

Purpose of this type class is following: given some type constructor `T[_]` (let's call it container) with arbitrary type `R` inside of this container, it will produce some output value of type `Out` as a result.

### Aux Pattern - why

Let's suppose we have some values of type `List[String]` and `List[Int]` in our program (but we are not limited to `List` we can use any type constructor, for example `Option`), then we can write generic method `extractor` to process these values by use of type class `Unwrap[T, R]` like this:

```scala
def extractor[T[_], R](in: T[R])(
  implicit
  unwrap: Unwrap[T, R]
): unwrap.Out = {
  unwrap(in)
}
```

It is worth noting how this method is implemented.

 1. It takes generic input parameter `in: T[R]` (our `List[String]` or `List[Int]`)
 2. Implicit parameter `unwrap: Unwrap[T, R]` is instance of our type class, which will be looked up in an implicit scope depending on the type parameters `T` and `R` of input value `in` (in our case `T` will be a `List` and `R` will be either a `String` or an `Int`).
 3. Return type of the method is **path dependent** type `unwrap.Out` (it depends solely on what implicit `unwrap: Unwrap[T, R]` value will be chosen by implicit search mechanism in point 2.).

We can provide instances of type class `Unwrap[T, R]` for `List[String]` and `List[Int]` in its companion object:

<a name="unwrap-typeclass-instances"></a>

```scala
object Unwrap {
  implicit object listStringSize extends Unwrap[List, String] {
    type Out = Int
    def apply(tr: List[String]): Int = tr.size
  }

  implicit object listIntMax extends Unwrap[List, Int] {
    type Out = Int
    def apply(tr: List[Int]): Int = tr.max
  }
}
```

Given these implicit values we can now use our `extractor` method. Given `List` of `Int`s, it returns maximal value, but given `List` of `String`s it returns size of the list itself.

```
scala> extractor(List(1,2,10,9))
res0: Unwrap.listIntMax.Out = 10

scala> extractor(List("1","2","10","9"))
res1: Unwrap.listStringSize.Out = 4
```

In both cases `Out` type is an `Int` but it can be an arbitrary type.

Up until now everything were working fine without any problems. Let's now add another feature to our `extractor` method. Let's say that aside from `unwrap.Out` return value itself we want also return `String` representation of this value. Let's define another type class for that purpose named `Printer[T]`.

```scala
trait Printer[T] {
  def apply(t: T): (String, T)
}
```

Definition of `Printer[T]` type class is trivial. It takes value of type `T` as a parameter and return `String` representation of a `T` together with `T` itself. We will define two instance of this type class. One for `T` being a `String` and one for `T` being an `Int`:

```scala
object Printer {
  implicit object stringPrinter extends Printer[String] {
    def apply(s: String): (String, String) = ("String: " + s, s)
  }

  implicit object intPrinter extends Printer[Int] {
    def apply(i: Int): (String, Int) = ("Int: " + i, i)
  }
}
```

Let's now use `Printer` type class in our `extractor` method:

```scala
def extractor[T[_], R](in: T[R])(
  implicit
  unwrap: Unwrap[T, R],
  withPrinter: Printer[unwrap.Out]
): (String, unwrap.Out) = {
  withPrinter(unwrap(in))
}
```

Note that we are now using `unwrap.Out` not only as a return type, but also as a type parameter to our `Printer` type class in `withPrinter: Printer[unwrap.Out]`. We need to do that since we want compiler to provide us proper instance of a `Printer` type class in dependence on types `T` and `R` of an input parameter `in`.

This looks promising, unfortunately it won't compile.

```
<console>:17: error: illegal dependent method type: parameter may only be referenced in a subsequent parameter section
         unwrap: Unwrap[T, R],
         ^
```

We've got some cryptic message from a compiler, but in an essence we hit a limitation in a Scala compiler how path dependent types and implicit values can interact. We need to came up with some workaround to overcome this limitation and this workaround is called `Aux pattern`.

## Aux Pattern to the rescue

We are going to look at two different implementation of an `Aux Pattern`. Let's give them names for purpose of this article:

  - *Explicit* - this is implementation of `Aux Pattern` used in Shapeless version 1.x.x. It is more verbose in comparison with version from Shapeless 2.x.x.
  - *Concise* - this is contemporary version of `Aux Pattern` you can find in Shapeless 2.x.x. It is more concise in comparison with Explicit version.

### Aux Pattern - Explicit implementation

As a first step of Explicit implementation we are going to define auxiliary type class called `UnwrapAux` which will be the same as the [`Unwrap` type class](#unwrap-typeclass) with the exception that the abstract type member `Out` will be removed and replaced by type parameter `Out` instead:

```scala
trait UnwrapAux[T[_], R, Out] {
  def apply(tr: T[R]): Out
}
```

Given this auxiliary type class `UnwrapAux` we can define implicit instances for it in companion object, similarly as we done before:

```scala
object UnwrapAux {
  implicit object listStringSize extends UnwrapAux[List, String, Int] {
    def apply(tr: List[String]): Int = tr.size
  }

  implicit object listIntMax extends UnwrapAux[List, Int, Int] {
    def apply(tr: List[Int]): Int = tr.max
  }
}
```

And surprisingly this is all we need to make `extractor` method work. We just use `UnwrapAux` in place of `Unwrap` and we add third type parameter `Out` to `extractor` method:

```scala
def extractor[T[_], R, Out](in: T[R])(
  implicit
  unwrap: UnwrapAux[T, R, Out],
  withPrinter: Printer[Out]
): (String, Out) = {
  withPrinter(unwrap(in))
}
```

Notice that in this implementation type `Out` is known beforehand so compiler knows what instance of `Printer` type class to search for.

But we are not done yet. Remember that in case of `extractor` implementation without `Printer` we returned path dependent type directly. Here is that implementation repeated once more:

```scala
def extractor[T[_], R](in: T[R])(
  implicit
  unwrap: Unwrap[T, R]
): unwrap.Out = {
  unwrap(in)
}
```

We don't need our auxiliary `UnwrapAux` type class here as `Unwrap` type class is sufficient enough in this case. But for this method to work with `Unwrap` type class we need implicit instances of it, but we have only implicit instances for `UnwrapAux` type class. One solution would be to define same implicit instances for `Unwrap` as we did for `UnwrapAux`, but that would lead to code duplication which we want to avoid. Another possibility is to reuse existing implicit `UnwrapAux` instances for defining implicit `Unwrap` instances.

We can achieve this by defining implicit method which will take implicit type class parameter `UnwrapAux[T, R, Out]` and we will use this parameter value to construct instance of `Unwrap[T, R]` type class. We will define this implicit method in `Unwrap` companion object.

<a name="manual-construction"></a>

```scala
object Unwrap {
  implicit def unwrap[T[_], R, Out0](
      implicit unwrapAux: UnwrapAux[T, R, Out0]) = new Unwrap[T, R] {
    type Out = Out0
    def apply(tr: T[R]): Out = unwrapAux(tr)
  }
}
```

Note that we need to use other name than `Out` for the name of a third type parameter in `UnwrapAux` type class as `Out` is a name of abstract type member of `Unwrap` type class itself. We can choose any name we like, but we will us name `Out0` to emphasise connection between `Out` and `Out0` types.

We just asked for implicit value of `unwrapAux: UnwrapAux[T, R, Out0]` and implemented `Unwrap[T, R]` by making abstract type member `Out` concrete by assigning it type parameter `Out0` and we implemented `def apply(tr: T[R])` method by passing input parameter `tr: T[R]` to `unwrapAux.apply` method.

And this is it. `Aux Pattern` with explicit conversion between `UnwrapAux` and `Unwrap`. I hope this will help you to understand how Concise version of `Aux Pattern` works.

### Aux Pattern - Concise implementation

This version is used in a Shapeless 2.x.x and is also described in a blog post mentioned at the beginning.

One notable difference in comparison with Explicit version is that this version doesn't have `UnwrapAux` type class at all.

We define implicit instances of the `Unwrap` type class as we did [at the beginning](#unwrap-typeclass-instances) ie. with type `Out` fixed to a concrete type. What we do extra though is to define auxiliary type `type Aux[T[_], R, Out0]` as an alias for refined type `Unwrap[T, R] { type Out = Out0 }`. Note that we are refining type `Out` in the same way as we did in Explicit implementation:

```scala
object Unwrap {

  type Aux[T[_], R, Out0] = Unwrap[T, R] { type Out = Out0 }

  implicit object listStringSize extends Unwrap[List, String] {
    type Out = Int
    def apply(tr: List[String]): Int = tr.size
  }

  implicit object listIntMax extends Unwrap[List, Int] {
    type Out = Int
    def apply(tr: List[Int]): Int = tr.max
  }
}
```

Usage of this concise implementation looks almost the same as usage with explicit `UnwrapAux` type class only with the difference that instead of `UnwrapAux[T, R, Out]` we will use `Unwrap.Aux[T, R, Out]` as a type for implicit value `unwrap`:

```scala
def extractor[T[_], R, Out](in: T[R])(
  implicit
  unwrap: Unwrap.Aux[T, R, Out],
  withPrinter: Printer[Out]
): (String, Out) = {
  withPrinter(unwrap(in))
}
```
If you are new to type refinement this imlementation may feel a little bit unclear. To make it more clear it can help to think in terms of Explicit implementation where we constructed auxiliary type [manually](#manual-construction). With time you will stop to worry about `Aux Pattern` workaround altogether and it became just natural building block to allow type dependent types to be used in single parameter block.

And that is all there is to `Aux Pattern`. I hope you've gained better familiarity with it by reading this blog post.

All code examples can be found on [Github](https://github.com/VlachJosef/aux-pattern).

Recommended sources: [Shapeless: Exploring Generic Programming in Scala](https://www.youtube.com/watch?v=GDbNxL8bqkY)
