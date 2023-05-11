---
sidebar_label: Typeclass Instances
title: Non-Orphan Typeclass Instances
---

As of smithy4s version `0.18.x` you are able to create custom typeclass instances inside the companion objects of classes in the code generated by smithy4s. This allows you to have instances that are found via implicit resolution without any need to special imports. Common examples where this will come in handy are for the `cats.Show` and `cats.Eq` typeclasses, but you can use this feature for any typeclass.

Here we will show an example using the `cats.Hash` typeclass.

## Setup Typeclass in Smithy

Here we will use the `smithy4s.meta#typeclass` trait to define a `hash` trait that represents the `cats.Hash` typeclass.

```smithy
use smithy4s.meta#typeclass

@trait
@typeclass(targetType: "cats.Hash", interpreter: "smithy4s.interopcats.SchemaVisitorHash")
structure hash {}
```

We are specifying `cats.hash` as the `targetType` since that is the typeclass which this trait represents. We are then specifying `smithy4s.interopcats.SchemaVisitorHash` as the classpath which points to a `CachedSchemaCompiler` for the `cats.Hash` typeclass.

## Implement CachedSchemaCompiler

Smithy4s has a concept called `CachedSchemaCompiler` which is an abstraction which we use here to interpret a `smithy4s.Schema` to produce an instance of a typeclass. Here is what this will look like:

```scala
object SchemaVisitorHash extends CachedSchemaCompiler.Impl[Hash] {
  protected type Aux[A] = Hash[A]
  def fromSchema[A](
      schema: Schema[A],
      cache: Cache
  ): Hash[A] = {
    schema.compile(new SchemaVisitorHash(cache))
  }
}
```

Here we are delegating to the `SchemaVisitorHash` which is doing the heavy lifting of interpreting the `smithy4s.Schema`. The `CachedSchemaCompiler.Impl` provides a `Cache` which we utilize to make sure we are not recompiling the same schema more than once. For more details on implementing a `SchemaVisitor`, you can check out the full `cats.Hash` schema compiler and visitor [here](https://github.com/disneystreaming/smithy4s/blob/series/0.18/modules/cats/src/smithy4s/interopcats/SchemaVisitorHash.scala).

## Use the hash typeclass trait

Now we are ready to use the `hash` trait we defined above.

```smithy
@hash
structure MovieTheater {
  name: String
}
```

This tells smithy4s to generate an instance of the `Hash` typeclass in the companion object of the `MovieTheater` type and to use the `CachedSchemaCompiler` defined above for the implementation. The generated code will look like:

```scala
case class MovieTheater(name: Option[String] = None)
object MovieTheater extends ShapeTag.Companion[MovieTheater] {
  val id: ShapeId = ShapeId("smithy4s.example", "MovieTheater")
 
  // ...

  implicit val schema: Schema[MovieTheater] = // ...

  implicit val movieTheaterHash: cats.Hash[MovieTheater] = SchemaVisitorHash.fromSchema(schema)
}
```