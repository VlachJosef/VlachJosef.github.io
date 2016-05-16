---
title: Scala Tagged types, Pt.1 - Introduction
date: 2016-05-08T11:15:28+01:00
layout: single
share: true
---

Tagged types - maybe you've heard about them, maybe you not, maybe you've even consider to use them, but decided not to bother in the end. Whatever the case in this article I will introduce them, explain what they are, tell why they are useful and show why to bother to use them.

Although I base all explanation on contrived example to show what advantages Tagged types brings us, this example is loosely based on my daily programmer routine.

## Problem ##

Let's imagine that we are building game server for mobile rpg game and one of game features is PvP battle. PvP battle allow players play against each other. Let's suppose that there are multiple PvP arenas in the game and that in every arena players will be placed to random bracket with another players who entered that arena. Goal of the player is to beat other players in his bracket. We focus on relations of `Arena`, `Bracket` and `Player` entities.

We can model this situation in Scala with case classes like this:

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
    /**
     * Change current bracket of player in arena
     */
    def changeBracket(arena: Arena, bracket: Bracket): PlayerProfile = {
        this.copy(bracketMapping = this.bracketMapping + (arena.id -> bracket.id))
    }
}

```

This is not only way how to model this situation, but we will use it for a sake of simplicity and convenience in this article as our driving model.

Now let's create state where there exist two arenas: *Fire Pit* and *Ice Dungeon*, where Fire Pit contains two brackets and Ice Dungeon contains single bracket, and let's create three players and put them into different arena brackets. Situation can then look like this:

```scala
val firePit    = Arena("firePit", "Fire Pit")
val iceDungeon = Arena("iceDungeon", "Ice Dungeon")
val bracket1 = Bracket("bracket1", "firePit")
val bracket2 = Bracket("bracket2", "firePit")
val bracket3 = Bracket("bracket3", "iceDungeon")
val player1 = PlayerProfile("player1", Map(firePit.id -> bracket1.id, iceDungeon.id -> bracket3.id))
val player2 = PlayerProfile("player2", Map(firePit.id -> bracket1.id))
val player3 = PlayerProfile("player3", Map(firePit.id -> bracket2.id, iceDungeon.id -> bracket3.id))
```

Let's focus our attention on the way how we defined value for field `bracketMapping` of `PlayerProfile`. For example in case of `player2` we defined its value in terms of `firePit` and `bracket1` instances like this `Map(firePit.id -> bracket1.id)`. Let's take a closer look on this.

Used like this everything looks clear and sound since we have everything basically on one screen, so every logical connection in our model is clearly visible, but with elapsed time and as the code will become more and more complex, definition like `bracketMapping` from `PlayerProfile` will gradually become unclear and confusing. When we would look on `bracketMapping` definition without any comment, how clear does it look to you?

```scala
bracketMapping: Map[String, String]
```

To me it doesn't seem clear at all. __What actually keys of that map represent? And what are values?__ Not to mention that actual name of a field `bracketMapping` doesn't help here too much.

On top of this there is even worse problem with such model definition. Can you spot an error in this implementation of `PlayerProfile` instance method `changeBracket`?

```scala
def changeBracket(arena: Arena, bracket: Bracket): PlayerProfile = {
    this.copy(bracketMapping = this.bracketMapping + (bracket.id -> arena.id))
}
```
We made a mistake and interchanged expression `bracket.id` with `arena.id`. We can see correct implementation in our [initial implementation](#model-no-tags). The worst thing about this bug is that __there were nothing to warn us when we made this mistake__. Code compile just fine.

These types of bugs result in weird errors, where things stop to work as expected and it is usually hard to figure out where the bug is. And the worst is that they are really easy to made.

So in our short code example we discovered two fundamental problems:

- Constructs like `Map[String, String]` are really hard to comprehend.
- Since every identifier is `String` it is easy to use wrong identifier (`bracket.id`) in place of another identifier (`arena.id`).

## Solution ##

Desirable solution should brings us these properties:

- We want to keep using `Map` data type as they are really convenient for our needs.
- We want compiler to catch improper use of `id`s for us (identifier of one class in place of identifier of second class).

To solve aforementioned problems and to get these properties we can use various techniques, one of which is to use Tagged types.

### Tagged types ###

Let's first look at how our model will look like with use of Tagged types:

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
    def changeBracket(arena: Arena, bracket: Bracket): PlayerProfile = {
        this.copy(bracketMapping = this.bracketMapping + (arena.id -> bracket.id))
    }
}
```

This is much easier to read and reason about. Even glance look at a type:

```scala
Map[ArenaId, BracketId]
```
is telling us what keys of this map represents and what values of this map stands for. The connection between keys and values of this `Map` and associated case classes is evident.

But what are these magical types `ArenaId`, `BracketId` and `ProfileId`, where are they coming from?

There is no magic here at all. Let's go step by step until we got to this implementation. As a first step we need to create some so called _tags_:

```scala
trait ProfileIdTag
trait ArenaIdTag
trait BracketIdTag
```

<a name="tagging-convention"></a>
As you can see, tags are ordinary traits without any implementation. In practice arbitrary type can be used as a tag, but traits are usually used for their convenience. The names also doesn't matter but it's good practice to use suffix like `...Tag` as a convention to distinguish tags from other types. We will see later why.

Now we are ready to use these tags (traits) to create Tagged types. For that we only need to import [@@](https://github.com/milessabin/shapeless/blob/shapeless-2.3.0-sjs-0.6.8/core/src/main/scala/shapeless/typeoperators.scala#L29) type definition from shapeless:

```scala
import shapeless.tag.@@
```

Now we can mark (tag) any type `T` simple like this:

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
    def changeBracket(arena: Arena, bracket: Bracket): PlayerProfile = {
        this.copy(bracketMapping = this.bracketMapping + (arena.id -> bracket.id))
    }
}
```
As you can see it is not so much different from our [initial implementation](#model-no-tags), but what differs significantly is the way how we use this new definition and advantages it brings us. But before we dive into usage, let's first tackle one problem this new definition has. This problem is unnecessary repetition of `String @@ ...` pattern in types declarations. To get rid of this repetition we can introduce simple type aliases for every single type `String @@ T`. In our example case we would need to define three type aliases:

```scala
type ProfileId = String @@ ProfileIdTag
type ArenaId   = String @@ ArenaIdTag
type BracketId = String @@ BracketIdTag
```

It is good practice to keep these type aliases short, since we will be using them more than tags itself. This is the reason why [above](#tagging-convention) we introduced that convention to use `..Tag` suffix for traits representing our tags.

With use of these type aliases we got our final definition as [shown above](#model-with-tags).

We will use these type aliases for the rest of this article.

### Usage ###

Now when we try to define `PlayerProfile` instance explicitly for example like this:

```scala
PlayerProfile("playerId", Map("firePit" -> "bracket1"))
```
it won't compile and we got two errors due to type mismatch:

```
[error] /Users/pepa/tagged-types/src/main/scala/io/vlach/tagged-pt1.scala:32: type mismatch;
[error]  found   : String("playerId")
[error]  required: io.vlach.tags.ProfileId
[error]   PlayerProfile("playerId", Map("firePit" -> "bracket1"))
[error]                 ^
[error] /Users/pepa/tagged-types/src/main/scala/io/vlach/tagged-pt1.scala:32: type mismatch;
[error]  found   : (String, String)
[error]  required: (io.vlach.tags.ArenaId, io.vlach.tags.BracketId)
[error]   PlayerProfile("playerId", Map("firePit" -> "bracket1"))
[error]                                           ^
[error] two errors found
```

These errors are telling us that we are trying to do something what we probably didn't mean to do. And really, we are using simple `String` types where Tagged types are expected. This can clearly be seen in the error messages.

To fix this problem we must make sure that parameters to `PlayerProfile.apply` method meet required tags criteria. So how can we actually create Tagged type from a `String`?


To turn `String` into Tagged type we use object [`tag`](https://github.com/milessabin/shapeless/blob/shapeless-2.3.0-sjs-0.6.8/core/src/main/scala/shapeless/typeoperators.scala#L25) from shapeless. Its usage is simple:

```scala
import shapeless.tag

val profileId: ProfileId = tag[ProfileIdTag][String]("profileId")
val arenaId: ArenaId     = tag[ArenaIdTag][String]("thePit")
val bracketId: BracketId = tag[BracketIdTag][String]("bracket1")
```

And that's it. Simple like this. We can use these values to create our `PlayerProfile` instance explicitly like this:

```scala
PlayerProfile(playerId, Map(arenaId -> bracketId))
```

and everything will work as expected. In case we accidentally swapped parameters for `arenaId` and `bracketId`:

```scala
PlayerProfile(playerId, Map(bracketId -> arenaId))
```

We would get compiler errors (output was simplified) telling us that we are doing something we didn't intended:

```
[error] /Users/pepa/tagged-types/src/main/scala/io/vlach/tagged-pt1.scala:38: type mismatch;
[error]  found   : (io.vlach.tags.BracketId, io.vlach.tags.ArenaId)
[error]  required: (io.vlach.tags.ArenaId, io.vlach.tags.BracketId)
[error]   PlayerProfile(playerId, Map(bracketId -> arenaId))
[error]                                         ^
[error] one error found
```

Not only Tagged types save us from wrong usage of values in our program, they even protect us agains wrong implementation. When we would try to implement `changeBracket` with `bracket.id` used instead of `arena.id` and vise versa as before:

```scala
def changeBracket(arena: Arena, bracket: Bracket): PlayerProfile = {
    this.copy(bracketMapping = this.bracketMapping + (bracket.id -> arena.id)) // won't compile
}
```

We would again receive compiler error warning us that our types don't match and thus saving us from potential long bug hunting later on.

I hope this article convince you that by using Tagged types you will gain lots of assistence from compiler helping you to avoid wide range of errors you would otherwise encounter at runtime or in better case have to cover by unit tests to prove correct implementation.

In second part of this article we will cover two main implementation of Tagged types in Scala. Namely Scalaz implementation and Shapeless implementation (used in this article).
