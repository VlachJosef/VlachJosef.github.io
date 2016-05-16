---
title: Practical Scala Pt1 - Reading spreadsheet data to case class
layout: single
---

When I got my first job as a full time scala developer in mobile game startup, one of my first task was to read data from google spreadsheet and use these data for constructing whole game configuration to one big case class. Let's try to follow my steps to tackle this problem from initial tries to final solution.

Let's imagine that our game configuration will need to load this data structure from google spreadsheet:

Cards table contains data about Card: `id`, `attack`, `defense`, `health`, `maxLevel`

CardCollections table contains: `id`, `cards`, `rewardId`

Rewards table contains: `id`, `rewardType`, `amount`, `rewardId`


```scala
case class CardTemplate(
	id: CardTemplateId,
	attack: Int,
	health: Int,
	maxLevel: Int,
	isBoss: Option[Boolean])
```

```scala
case class CardCollection(
	id: CardCollectionId,
	cards: List[CardTemplate],
	rewardId: RewardData)
```

```scala
case class RewardData(
	id: RewardDataId,
	rewardType: RewardType,
	amount: Int,
	rewardId: Option[CardTemplateId])

```

I will base this blog on following simplified configuration, which I will represent as `Map[String, String]` to allow us not to bother with spreadsheet reading which in reality would be done with [Google Sheets API][].




```scala
package object pickling {
  import scala.collection.immutable.Map
  import scala.pickling._
  import scala.language.implicitConversions

  implicit def pickleFormat: MapPickleFormat = new MapPickleFormat()

  implicit def toUnpickleOps(value: Map[String, String]): UnpickleOps = new UnpickleOps(MapPickle(value))
}
```

```scala
package pickling {
  import scala.pickling._

  case class MapPickle(value: Map[String, String]) extends Pickle {
    type ValueType = Map[String, String]
    type PickleFormatType = MapPickleFormat
  }

  case object NonExistentProperty

  class MapPickleFormat extends PickleFormat {

    type PickleType = MapPickle
    type OutputType = Output[String]

    def createBuilder() = new MapPickleBuilder(this, new StringOutput)

    def createBuilder(out: OutputType): PBuilder = new MapPickleBuilder(this, out)

    def createReader(pickle: MapPickle): PReader = new MapPickleReader(pickle.value, this)

    val OptionPattern = "scala.Option\\[(.*)\\]".r
    val SomePattern = "scala.Some\\[(.*)\\]".r

    class MapPickleBuilder(format: MapPickleFormat, buf: OutputType) extends PBuilder with PickleTools {

      private val results = scala.collection.mutable.Map.empty[String, String]
      private var isSome = false
      private var skipNone = false
      private var lastValue: String = ""

      override def beginEntry(picklee: Any): PBuilder = withHints { hints =>
        //println("==============================================")
        //println("beginEntry: " + picklee)
        //println("beginEntry: hints.isElidedType " + hints.isElidedType)
        //println("beginEntry: hints.tag          " + hints.tag)
        //println("beginEntry: hints.tag.key      " + hints.tag.key)

        val key = hints.tag.key

        key match {
          case SomePattern(c) => isSome = true
          case "scala.None.type" => skipNone = true
          case _ => lastValue = picklee.toString
        }

        this
      }

      override def endEntry(): Unit = {
        //println("endEntry")
      }

      override def putField(name: String, pickler: (PBuilder) => Unit): PBuilder = {
        println("putField(" + name + ")")
        pickler(this)

        if(skipNone) {
          skipNone = false
        } else if(isSome) {
          isSome = false
        } else {
          results += name -> lastValue
        }

        this
      }

      override def result(): MapPickle = {
        //println("result")
        MapPickle(results.toMap)
      }

      override def putElement(pickler: (PBuilder) => Unit): PBuilder = {
        //println("putElement")
        this
      }

      override def beginCollection(length: Int): PBuilder = {
        //println("beginCollection")
        this
      }

      override def endCollection(): Unit = {
        //println("endCollection")
      }
    }

    class MapPickleReader(var datum: Any, format: MapPickleFormat) extends PReader with PickleTools {

      println("CREATING NEW MapPickleFormat " + datum)

      private var lastReadTag: String = null

      private val primitives = Map[String, () => Any](
        FastTypeTag.Unit.key -> (() => ()),
        FastTypeTag.Null.key -> (() => null),
        FastTypeTag.Int.key -> (() => {
          datum match {
            case NonExistentProperty => 0
            case i: Int => i
            case s: String => s.toInt
          }
        }),
        FastTypeTag.Short.key -> (() => datum.asInstanceOf[Double].toShort),
        FastTypeTag.Double.key -> (() => {
          datum match {
            case NonExistentProperty => 0.0
            case d: Double => d
            case s: String => s.toDouble
          }
        }),
        FastTypeTag.Float.key -> (() => datum.asInstanceOf[Double].toFloat),
        FastTypeTag.Long.key -> (() => {
          datum match {
            case NonExistentProperty => 0L
            case l: Long => l
            case s: String => s.toLong
          }
        }),
        FastTypeTag.Byte.key -> (() => datum.asInstanceOf[Double].toByte),
        FastTypeTag.Boolean.key -> (() => {
          datum match {
            case NonExistentProperty => false
            case b: Boolean => b
            case s: String => s.toBoolean
            case null => false
          }
        }),
        FastTypeTag.Char.key -> (() => datum.asInstanceOf[String].head),
        FastTypeTag.String.key -> (() => {
          datum match {
            case NonExistentProperty => ""
            case null => ""
            case _ => datum.asInstanceOf[String]
          }
                                   })
        //FastTypeTag.List.key -> (() => ???)
      )

      override def beginEntry(): String = withHints { hints =>
        //println("==============================================")
        println("beginEntry: " + datum)
        println("beginEntry: hints.isElidedType " + hints.isElidedType + " " + (datum == null))
        println("beginEntry: hints.tag          " + hints.tag)
        println("beginEntry: hints.tag.key          " + hints.tag.key)
        println("hints.tag.tpe " + hints.tag.tpe)

        lastReadTag =
          if (hints.isElidedType) {
            datum match {
              case _ => hints.tag.key
            }
          } else {
            println("datum " + datum)
            datum match {
              case NonExistentProperty => "scala.None.type"
              case map: Map[String, String] => hints.tag.key
              case null => "scala.None.type"
              case some =>
                hints.tag.key match {
                  case OptionPattern(c) => "scala.Some[" + c + "]"
                  case _ => "scala.Some[java.lang.String]" // We should not take this branch
                }
            }
          }
        println("lastReadTag " + lastReadTag)
        lastReadTag
      }

      override def readPrimitive(): Any = {

        println("readPrimitive START lastReadTag " + lastReadTag)
        val res = primitives(lastReadTag)()
        println("readPrimitive res=" + res)
        res
      }

      override def beginCollection(): PReader = {
        println("beginCollection")
        ???
      }

      override def readField(name: String): PReader = {
        println("readField name " + name)
        //println("readField datum " + datum)
        val value = datum match {
          case map: Map[String, String] =>
            map.getOrElse(name, NonExistentProperty)
          case some => some
        }
        //println("readField value " + value)
        //println("readField value " + (datum == null))
        //println("readField name -> value " + name + " -> " + value)

        new MapPickleReader(value, format)
      }

      override def endEntry(): Unit = {
        println("endEntry")
      }

      override def atPrimitive: Boolean = {

        val res = primitives.contains(lastReadTag)
        println("atPrimitive res " + res + " for "+ lastReadTag)
        res
      }

      override def readElement(): PReader = {
        println("readElement")
        ???
      }

      override def readLength(): Int = {
        println("readLength")
        ???
      }

      override def atObject: Boolean = {
        println("atObject")
        ???
      }

      override def endCollection(): Unit = {
        println("endCollection")
        ???
      }
    }
  }
}

```

[Google Sheets API]: https://developers.google.com/google-apps/spreadsheets/
