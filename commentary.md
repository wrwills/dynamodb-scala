### Commentary on Code


In terms of code style, I don't find this displeasing.  I have put a few comments in various places in the codebase,
 and they can be seen by diffing my branch against master.

I suppose the main thing that struck me about the code in this project is that it seemed to require a fair amount of
boilerplate to set up.



I believe that one issue with dynamo db is that it doesn't allow for multi column keys.  One way around this is to
make a tuple a primary key.  However this is not possible:

```scala
scala> Key[(String,String)]("seq")
<console>:14: error: could not find implicit value for parameter encoder: com.onzo.dynamodb.Encoder[(String, String)]
       Key[(String,String)]("seq")
```

I suspect it could be useful to add some type instances for Encoder.


Doesn't appear to allow for nesting objects.  eg in the tests GameScore is defined as follows: 

```scala
case class GameScore(
                      userId: String,
                      gameTitle: String,
                      topScore: Long,
                      topScoreDateTime: DateTime,
                      wins: Long,
                      losses: Long,
                      extra: Map[String, String] = Map.empty,
                      seq: Seq[String] = Seq.empty,
                      optThing: Option[Boolean] = None,
                      mapKey : Map[String, String] = Map.empty
                    )
```

I can imagine for certain use cases one might want to have nested classes. eg 

```scala
case class Foo(a: String)
case class Bar(b: String, foo: Foo)
```

and to be able to use tho sewith DynamoDb, but the way TableMapper and Key work  makes this difficult:

```scala
scala> Key[Foo]("seq")
<console>:16: error: could not find implicit value for parameter encoder: com.onzo.dynamodb.Encoder[Foo]
```

Another project (written by someone at the Guardian), Scanamo, is able to nest objects as described.  I haven't 
had time to fully understand how it works, but it appears to be using typeclasses in a way similar
to how most Scala json libraries work these days, but instead of producing json it's producing AttributeValues.

I get the feeling that perhaps there should be a way to generate TableMapper directly from case classes rather than going 
from the TableMapper to the case class.

```scala
val * : TableMapper[GameScore] = {
      PrimaryKey[String]("UserId") ::
        RangeKey[String]("GameTitle") ::
        Key[Long]("TopScore") ::
        Key[DateTime]("TopScoreDateTime")(Encoder[String].contramap { d: DateTime => fmt.print(d) }, Decoder[String].map(fmt.parseDateTime)) ::
        Key[Long]("Wins") ::
        Key[Long]("Losses") ::
        Key[Map[String, String]]("extra") ::
        Key[Seq[String]]("seq") ::
        Key[Option[Boolean]]("optThing") ::
        MapKey[String] :: HNil
    }.as[GameScore]
```

Perhaps Shapeless labelled generics could be adapted to generate TableMapper objcets from case classes?  Eg in the shapeless
documentation there is the following example:

```scala
val gen = LabelledGeneric[Person]
val joeRecord = gen.to(joe)
joeRecord.values
res1: shapeless.::[String,shapeless.::[String,shapeless.::[Int,shapeless.HNil]]] 
```

There should be a way to have something similar to gen that instead produced and hlist of keys, perhaps specifying
a primary keys etc.

Anothter approach that occurs to me would be to use something more like Quill which uses macros to map case classes to 
datastores.  Quill does have an implementation for Cassandra.  Perhaps it could be adapted?

