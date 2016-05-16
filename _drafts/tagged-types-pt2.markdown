---
title: Scalaz vs Shapeless tags, Pt.2 - Comparison
layout: single
share: true
---

In [first](http://localhost:4000/tagged-types-pt1/) part of this series we introduced Tagged types and show how they improve code quality. In this part of series we will look at two main implementation of Tagged types and compare their usage.




I would like to introduce Scala Tagged types, what are they, why are they useful and in second part of this series I will compare two main implementation in Scala.

I will base all explanation on 'almost' real world example and show what advantage Tagged types brings us in everyday programming.

## Problem ##

Let's imagine that we are building game server for mobile rpg game and one of game features is PvP battle. PvP battle allow players play against each other. Let's suppose that there are multiple PvP arenas in the game and that in every arena player will be placed to random bracket with another players who entered that arena. Goal of the player is to beat other players in his bracket. We focus on relations of `Arena`, `Bracket` and `Player` entities.

Simplified model of this situation defined in case classes could looks like this:

<a name="model-no-tags"></a>

```scala
case class Arena(
	id: String, // unique identifier of the arena
	name: String // name of the arena
)

case class Bracket(
	id: String, // unique identifier of pvp bracket
	arenaId: String // to which arena this bracket belongs
)

case class PlayerProfile(
	id: String, // unique player profile identifier
	bracketMapping: Map[String, String] // arena to bracket mapping
) {
    def changeBracket(arenaId: String,
                      bracketId: String): PlayerProfile = {
    this.copy(bracketMapping = this.bracketMapping + (arenaId -> bracketId))
  }
}
```

This is not only way how to model this, but we will use it for a sake of simplicity in this post as an example.

Now let's create state where there exist two arenas: *Fire Pit* and *Ice Dungeon*, where Fire Pit contains two brackets and Ice Dungeon contains single bracket, and let's create three players and put them into different arena brackets. Situation then can look like this:

```scala
val firePit    = Arena("firePit", "Fire Pit", 10)
val iceDungeon = Arena("iceDungeon", "Ice Dungeon", 20)
val bracket1 = Bracket("bracket1", "firePit")
val bracket2 = Bracket("bracket2", "firePit")
val bracket3 = Bracket("bracket3", "iceDungeon")
val p1 = PlayerProfile("p1", Map("firePit" -> "bracket1", "iceDungeon" -> "bracket3"))
val p2 = PlayerProfile("p2", Map("firePit" -> "bracket1"))
val p3 = PlayerProfile("p3", Map("firePit" -> "bracket2", "iceDungeon" -> "bracket3"))
```

We can see that in `bracketMapping` field of `PlayerProfile` (for instance in: `Map("firePit" -> "bracket1")`) there are `id`s from `Arena` and `Bracket` case class instances. I would like to bring your attention to this field.

Now everything looks clear and sound since we have everything defined basically on one screen, so every connection in our model is clearly visible, but with elapsed time and as the code will become more and more complex, definition like `bracketMapping` from `PlayerProfile` will gradually become unclear and confusing. When we would look on `bracketMapping` definition without any comment, how clear does it looks to you?

```scala
bracketMapping: Map[String, String]
```

To me it doesn't seem clear at all. __What actually keys of that map represent? And what are values?__ Not to mention that actual name of a field `bracketMapping` doesn't help here too much.

On top of this there is even worse problem with such model definition. Can you spot an error in this implementation of `PlayerProfile` instance method `changeBracket`?

```scala
def changeBracket(arenaId: String, bracketId: String): PlayerProfile = {
	this.copy(bracketMapping = this.bracketMapping + (bracketId -> arenaId))
}
```
We made a mistake and interchanged `bracketId` with `arenaId`. Correct implementaton looks like in our [initial implementation](#model-no-tags). The worst thing about this bug is that __there were nothing to warn us when we made this mistake__. Code compile just fine.

These types of bugs results in weird errors, where things stop to work as expected and it is usually hard to figure out where the bug is.

So in our short code example we discovered two fundamental problems:

- Constructs like `Map[String, String]` are really hard to comprehend.
- Since every identifier is `String` it is easy to use wrong identifier (`bracketId`) in place if another identifier (`arenaId`).

## Solution ##

Desirable solution should brings us these properties:

- We want to keep using `Map` data type as they are really convenient for our needs.
- We want compiler to catch improper use of `id`s for us.

To solve aforementioned problems and to get these properties we can use various techniques, one of which is to use Tagged types.

### Tagged types ###

Let's first look at how our model will look like with Tagged types:

<a name="model-with-tags"></a>

```scala
case class Arena(
	id: ArenaId, // <- tagged type
	name: String
)

case class Bracket(
	id: BracketId, // <- tagged type
	arenaId: ArenaId // <- tagged type
)

case class PlayerProfile(
	id: ProfileId, // <- tagged type
	bracketMapping: Map[ArenaId, BracketId] // <- tagged types
) {
    def changeBracket(arenaId: ArenaId,
                      bracketId: BracketId): PlayerProfile = {
    this.copy(bracketMapping = this.bracketMapping + (arenaId -> bracketId))
  }
}
```

This is much easier to read and reason about. Even glance look at a type:

```scala
Map[ArenaId, BracketId]
```
is telling us what keys of this map represents and what values of this map stands for. The connection between keys and values of this `Map` and associated case classes is evident.

But what are these magical types `ArenaId`, `BracketId` and `ProfileId`, where are they coming from?

There is no magic here at all. Let's go step by step until we got to this implementation. As a first step we need to create some _tags_:

```scala
trait ArenaIdTag
trait BracketIdTag
trait ProfileIdTag
```

<a name="tagging-convention"></a>
As you can see, tags are ordinary traits without any implementation. In practice arbitrary type can be used as a tag, but traits are usually used for their convenience. The names also doesn't matter but it's good practice to use suffix like `...Tag` as a convention. We will see later why.

Now we are ready to use these traits to create Tagged types. For that we only need to import [@@](https://github.com/milessabin/shapeless/blob/shapeless-2.3.0-sjs-0.6.8/core/src/main/scala/shapeless/typeoperators.scala#L29) type definition from shapeless:

```scala
import shapeless.tag.@@
```

Now we can tag any type `T` simple like this:

```scala
T @@ OurIdTag
```

For example when we want to tag `String` field of some case class we do it like this: `String @@ OurIdTag`. We can tag arbitrary type not just `String`, so if we need to tag `Int` field of some case class we can do it like this: `Int @@ OurIdTag`.

Let's take a look at a definition of `PlayerProfile` case class with `changeBracket` method when we use tagged types:

```scala
case class PlayerProfile(
  id: String @@ ProfileIdTag,
  bracketMapping: Map[String @@ ArenaIdTag, String @@ BracketIdTag]
) {
    def changeBracket(arenaId: String @@ ArenaIdTag,
                      bracketId: String @@ BracketIdTag): PlayerProfile = {
    this.copy(bracketMapping = this.bracketMapping + (arenaId -> bracketId))
  }
}
```
As you can see it is not so much different from our [initial implementation](#model-no-tags), but what differs significantly is the way how we use this new definition and advantages it brings us. But before we dive into usage, let's first tackle one problem this new definition has. This problem is unnecessary repetition of `String @@ ...` pattern in types declarations. To get rid of this repetition we can introduce simple type aliases for every single type `String @@ T`. In our example case we would need to define three type aliases:

```scala
type ArenaId   = String @@ ArenaIdTag
type BracketId = String @@ BracketIdTag
type ProfileId = String @@ ProfileIdTag
```

It is good practice to keep these type aliases short, since we will be using them everywhere. This is the reason why [above](#tagging-convention) we introduced that convention to use `..Tag` suffix for traits representing our tags.

With use of these type aliases we got our final definition as [shown above](#model-with-tags).

We will use these type aliases for the rest of this article.

### Usage ###

Now when we try to call method `changeBracket` with `String` parameters:

```scala
changeBracket("thePit", "bracket1")
```

it won't compile due to type mismatch:

```
[error] /Users/pepa/blog-post-1/src/main/scala/io/foldright/tags.scala:57: type mismatch;
[error]  found   : String("thePit")
[error]  required: shapeless.tag.@@[String, ArenaIdTag]
...
[error] /Users/pepa/blog-post-1/src/main/scala/io/foldright/tags.scala:57: type mismatch;
[error]  found   : String("bracket1")
[error]  required: shapeless.tag.@@[String, BracketIdTag]

```
These errors are telling us that we are trying to do something what we probably didn't mean to do. And really, we are using simple `String`s in place where Tagged types are expected. To fix this problem we must make sure that parameters to `changeBracket` method meet required tags. So how can we actually create tagged type from a `String`?


To turn `String` into Tagged type we use object [`tag`](https://github.com/milessabin/shapeless/blob/shapeless-2.3.0-sjs-0.6.8/core/src/main/scala/shapeless/typeoperators.scala#L25) from shapeless. Its usage is simple:

```scala
import shapeless.tag

val arenaId:   ArenaId   = tag[ArenaIdTag][String]("thePit")
val bracketId: BracketId = tag[BracketIdTag][String]("bracket1")
```

And that's it. Simple like this. We can use these types to call our `changeBracket` method

```scala
changeBracket(arenaId, bracketId)
```

and everything will work as expected. In case we accidentally swapped parameters for `arenaId` and `bracketId`:

```scala
changeBracket(bracketId, arenaId)
```

We would get compiler errors (output was simplified) telling us that we are doing something we didn't intended:

```
[error] /Users/pepa/blog-post-1/src/main/scala/foldright/tags.scala:65: type mismatch;
[error]  found   : shapeless.tag.@@[String, BracketIdTag]
[error]  required: shapeless.tag.@@[String, ArenaIdTag]
[error]   changeBracket(bracketId, arenaId)
[error]                  ^
[error] /Users/pepa/blog-post-1/src/main/scala/foldright/tags.scala:65: type mismatch;
[error]  found   : shapeless.tag.@@[String, ArenaIdTag]
[error]  required: shapeless.tag.@@[String, BracketIdTag]
[error]   changeBracket(bracketId, arenaId)
[error]                              ^
```

Not only Tagged types save us from wrong usage, they even protect us agains wrong implementation. When we would try to implement `changeBracket` with `bracketId` used instead of `arenaId` and vise versa as before:

```scala
def changeBracket(arenaId: ArenaId,
                  bracketId: BracketId): PlayerProfile = {
  this.copy(bracketMapping = this.bracketMapping + (bracketId -> arenaId))
}
```

We would again receive compiler error warning us that our types don't match and thus saving us from potential long bug hunting later on.

I hope this article convince you that by using Tagged types you will gain lots of help from compiler helping you to avoid wide range of errors you would otherwise encounter at runtime or in better case have to cover by unit tests to prove correct implementation.

In second part of this article we will cover two main implementation of Tagged types in Scala.
