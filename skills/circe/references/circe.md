# Circe Reference

## Table of Contents
- [Sources](#sources)
- [Dependencies](#dependencies)
- [Core Types](#core-types)
- [Parsing JSON](#parsing-json)
- [Encoding and Decoding Basics](#encoding-and-decoding-basics)
- [Cursor Traversal and Modification](#cursor-traversal-and-modification)
- [Semi-automatic Derivation](#semi-automatic-derivation)
- [Automatic Derivation](#automatic-derivation)
- [Custom Codecs](#custom-codecs)
- [ADT Encoding and Decoding](#adt-encoding-and-decoding)
- [Recursive ADTs](#recursive-adts)
- [Optics](#optics)
- [Configuration and Extras](#configuration-and-extras)
- [Common Pitfalls and FAQ](#common-pitfalls-and-faq)
- [Test Prompts](#test-prompts)

---

## Sources

- Official docs: https://circe.github.io/circe/
- GitHub: https://github.com/circe/circe
- Scaladoc: https://www.javadoc.io/doc/io.circe/circe-core_3/latest/index.html

---

## Dependencies

```scala
val circeVersion = "0.14.15"

libraryDependencies ++= Seq(
  "io.circe" %% "circe-core"           % circeVersion,
  "io.circe" %% "circe-generic"        % circeVersion,  // semi-auto & auto derivation
  "io.circe" %% "circe-parser"         % circeVersion,  // JSON parsing (Jawn on JVM)
  "io.circe" %% "circe-generic-extras" % circeVersion,  // Configuration, @ConfiguredJsonCodec, etc.
  "io.circe" %% "circe-optics"         % "0.15.0",      // Monocle-based optics
  "io.circe" %% "circe-shapes"         % circeVersion   // Shapeless Coproduct support
)
```

The core project depends only on `cats-core`. Parser, generic derivation, and optics are separate modules.

---

## Core Types

| Type | Purpose |
|---|---|
| `Json` | Immutable JSON AST. Constructors: `Json.obj(...)`, `Json.arr(...)`, `Json.fromString(...)`, `Json.fromInt(...)`, `Json.Null`, `Json.fromBoolean(...)` |
| `Encoder[A]` | Type class converting `A` to `Json` |
| `Decoder[A]` | Type class converting `Json` to `Either[DecodingFailure, A]` |
| `Codec[A]` | Combines `Encoder[A]` and `Decoder[A]` |
| `HCursor` | Cursor with history — primary way to traverse and extract from JSON |
| `ACursor` | Cursor that may represent a failed navigation |
| `KeyEncoder[A]` | Encodes `A` as a JSON object key (String) |
| `KeyDecoder[A]` | Decodes a JSON object key (String) to `A` |
| `ParsingFailure` | Error from parsing invalid JSON |
| `DecodingFailure` | Error from decoding JSON to a Scala type |

---

## Parsing JSON

```scala
import io.circe._, io.circe.parser._

// Parse a JSON string
val rawJson: String = """
  {
    "foo": "bar",
    "baz": 123,
    "list of stuff": [ 4, 5, 6 ]
  }
"""

val parseResult: Either[ParsingFailure, Json] = parse(rawJson)

// Pattern matching on result
parse(rawJson) match {
  case Left(failure) => println("Invalid JSON :(")
  case Right(json)   => println("Yay, got some JSON!")
}

// Using getOrElse (Cats extension method)
val json: Json = parse(rawJson).getOrElse(Json.Null)

// Direct parse + decode in one step
import io.circe.parser.decode
decode[List[Int]]("[1, 2, 3]")
// res: Either[io.circe.Error, List[Int]] = Right(List(1, 2, 3))
```

**Invalid JSON returns Left:**
```scala
parse("yolo")
// Left(ParsingFailure("expected json value got 'yolo' (line 1, column 1)", ...))
```

---

## Encoding and Decoding Basics

```scala
import io.circe.syntax._

// Encoding
val intsJson = List(1, 2, 3).asJson
// JArray(Vector(JNumber(1), JNumber(2), JNumber(3)))

// Decoding from Json
intsJson.as[List[Int]]
// Right(List(1, 2, 3))

// Direct string decoding
import io.circe.parser.decode
decode[List[Int]]("[1, 2, 3]")
// Right(List(1, 2, 3))
```

Built-in `Encoder`/`Decoder` instances exist for: `Int`, `Long`, `Double`, `Float`, `String`, `Boolean`, `BigDecimal`, `BigInt`, `Option[A]`, `List[A]`, `Vector[A]`, `Set[A]`, `Map[String, A]`, `Either[A, B]`, tuples, `java.util.UUID`, and more.

---

## Cursor Traversal and Modification

### Setup
```scala
import cats.syntax.either._
import io.circe._, io.circe.parser._

val json: String = """
  {
    "id": "c730433b-082c-4984-9d66-855c243266f0",
    "name": "Foo",
    "counts": [1, 2, 3],
    "values": {
      "bar": true,
      "baz": 100.001,
      "qux": ["a", "b"]
    }
  }
"""

val doc: Json = parse(json).getOrElse(Json.Null)
val cursor: HCursor = doc.hcursor
```

### Extracting values
```scala
// Navigate to nested field and decode
val baz: Decoder.Result[Double] =
  cursor.downField("values").downField("baz").as[Double]

// Shorthand with .get
val baz2: Decoder.Result[Double] =
  cursor.downField("values").get[Double]("baz")

// Navigate into array (first element)
val secondQux: Decoder.Result[String] =
  cursor.downField("values").downField("qux").downArray.as[String]
```

### Cursor navigation methods
| Method | Purpose |
|---|---|
| `downField("key")` | Move to a field in a JSON object |
| `downArray` | Move to the first element of a JSON array |
| `downN(n)` | Move to the nth element of a JSON array |
| `get[T]("key")` | Shorthand for `downField("key").as[T]` |
| `as[T]` | Decode the current focus |
| `focus` | Get the current `Json` value as `Option[Json]` |
| `top` | Get the root `Json` with all modifications applied |
| `up` | Move up one level |
| `left` / `right` | Move to sibling in array |
| `keys` | Get field names of current JSON object |
| `values` | Get values of current JSON object |

### Modifying JSON
```scala
// Transform a field value
val reversedNameCursor: ACursor =
  cursor.downField("name").withFocus(_.mapString(_.reverse))

// Get the full modified document
val reversedName: Option[Json] = reversedNameCursor.top
```

---

## Semi-automatic Derivation

**Recommended approach for production code.** Explicit, no surprises, good compile times.

### Basic pattern
```scala
import io.circe._, io.circe.generic.semiauto._, io.circe.syntax._

case class Foo(a: Int, b: String, c: Boolean)

implicit val fooDecoder: Decoder[Foo] = deriveDecoder[Foo]
implicit val fooEncoder: Encoder[Foo] = deriveEncoder[Foo]

// Type parameter can be inferred:
implicit val fooDecoder: Decoder[Foo] = deriveDecoder
implicit val fooEncoder: Encoder[Foo] = deriveEncoder

Foo(13, "Qux", false).asJson
// {"a": 13, "b": "Qux", "c": false}
```

### @JsonCodec annotation
```scala
import io.circe.generic.JsonCodec, io.circe.syntax._

@JsonCodec case class Bar(i: Int, s: String)

Bar(13, "Qux").asJson
// {"i": 13, "s": "Qux"}
```
Requires compiler flag `-Ymacro-annotations` (Scala 2.13+) or Macro Paradise plugin (Scala 2.10-2.12).

### forProductN helpers (no shapeless dependency)
```scala
import io.circe.{ Decoder, Encoder }

case class User(id: Long, firstName: String, lastName: String)

implicit val decodeUser: Decoder[User] =
  Decoder.forProduct3("id", "first_name", "last_name")(User.apply)

implicit val encodeUser: Encoder[User] =
  Encoder.forProduct3("id", "first_name", "last_name")(u =>
    (u.id, u.firstName, u.lastName)
  )
```
Only requires `circe-core`. Best for custom field name mapping. Available from `forProduct1` to `forProduct22`.

### Unwrapped value classes (circe-generic-extras)
```scala
import io.circe._, io.circe.generic.extras.semiauto._

case class Foo(a: Int)

implicit val fooDecoder: Decoder[Foo] = deriveUnwrappedDecoder[Foo]
implicit val fooEncoder: Encoder[Foo] = deriveUnwrappedEncoder[Foo]

// Foo(123) serializes as: 123  (not {"a": 123})
```

---

## Automatic Derivation

**Use for prototyping only.** Implicit scope pollution, slower compile times, harder to debug.

```scala
import io.circe.generic.auto._, io.circe.syntax._

case class Person(name: String)
case class Greeting(salutation: String, person: Person, exclamationMarks: Int)

Greeting("Hey", Person("Chris"), 3).asJson
// {"salutation": "Hey", "person": {"name": "Chris"}, "exclamationMarks": 3}
```

The single import `io.circe.generic.auto._` derives `Encoder` and `Decoder` for any case class or sealed trait hierarchy in implicit scope via Shapeless.

**Caveat:** If you use `-Ypartial-unification` and auto derivation, incomplete decoders will not work (see circe #724).

---

## Custom Codecs

### Encoder.instance / Decoder.instance
```scala
import io.circe.{ Decoder, Encoder, HCursor, Json }

class Thing(val foo: String, val bar: Int)

implicit val encodeThing: Encoder[Thing] = Encoder.instance { a =>
  Json.obj(
    ("foo", Json.fromString(a.foo)),
    ("bar", Json.fromInt(a.bar))
  )
}

implicit val decodeThing: Decoder[Thing] = Decoder.instance { c =>
  for {
    foo <- c.downField("foo").as[String]
    bar <- c.get[Int]("bar")
  } yield new Thing(foo, bar)
}
```

### Accumulating error decoder
```scala
import cats.syntax.all._

implicit val decodeThingAcc: Decoder[Thing] = Decoder.accumulatingInstance { c =>
  (
    c.downField("foo").asAcc[String],
    c.getAcc[Int]("bar")
  ).mapN { (foo, bar) =>
    new Thing(foo, bar)
  }
}
```
Collects all decoding errors instead of failing on the first one.

### Custom type codec (e.g., java.time.Instant)
```scala
import io.circe.{ Decoder, Encoder }
import java.time.Instant
import scala.util.Try

implicit val encodeInstant: Encoder[Instant] =
  Encoder.encodeString.contramap[Instant](_.toString)

implicit val decodeInstant: Decoder[Instant] =
  Decoder.decodeString.emapTry { str =>
    Try(Instant.parse(str))
  }
```

### Decoder transformation methods
| Method | Purpose |
|---|---|
| `emap(f: String => Either[String, A])` | Map with possible failure (from String) |
| `emapTry(f: String => Try[A])` | Map with Try |
| `validate(pred, msg)` | Add validation |
| `or(other)` | Try this decoder, fall back to other |
| `at("field")` | Decode value at a specific field |
| `prepare(f: ACursor => ACursor)` | Transform cursor before decoding |

### Encoder transformation methods
| Method | Purpose |
|---|---|
| `contramap[B](f: B => A)` | Adapt encoder for a new input type |
| `mapJson(f: Json => Json)` | Transform the output JSON |

### Custom key types for Map
```scala
import io.circe._, io.circe.syntax._

case class Foo(value: String)

implicit val fooKeyEncoder: KeyEncoder[Foo] =
  KeyEncoder.instance[Foo](_.value)

implicit val fooKeyDecoder: KeyDecoder[Foo] =
  KeyDecoder.instance[Foo](key => Some(Foo(key)))

// Now Map[Foo, Int] can be encoded/decoded
val map = Map(Foo("a") -> 1, Foo("b") -> 2)
map.asJson  // {"a": 1, "b": 2}
```

---

## ADT Encoding and Decoding

### ADT definition
```scala
sealed trait Event
case class Foo(i: Int) extends Event
case class Bar(s: String) extends Event
case class Baz(c: Char) extends Event
case class Qux(values: List[String]) extends Event
```

### Manual ADT codec (explicit, most control)
```scala
import cats.syntax.functor._
import io.circe.{ Decoder, Encoder }, io.circe.generic.auto._
import io.circe.syntax._

implicit val encodeEvent: Encoder[Event] = Encoder.instance {
  case foo: Foo => foo.asJson
  case bar: Bar => bar.asJson
  case baz: Baz => baz.asJson
  case qux: Qux => qux.asJson
}

implicit val decodeEvent: Decoder[Event] =
  List[Decoder[Event]](
    Decoder[Foo].widen,
    Decoder[Bar].widen,
    Decoder[Baz].widen,
    Decoder[Qux].widen
  ).reduceLeft(_ or _)
```

**Important:** `.widen` is needed because circe's type classes are invariant. It upcasts `Decoder[Foo]` to `Decoder[Event]`.

**Decoder ordering matters:** decoders are tried in order. If two case classes have overlapping fields, the first match wins.

### Usage
```scala
import io.circe.parser.decode

decode[Event]("""{ "i": 1000 }""")
// Right(Foo(i = 1000))

(Foo(100): Event).asJson.noSpaces
// {"i":100}
```

### With discriminator field (circe-generic-extras)
```scala
import io.circe.generic.extras.auto._
import io.circe.generic.extras.Configuration

implicit val genDevConfig: Configuration =
  Configuration.default.withDiscriminator("what_am_i")

(Foo(100): Event).asJson.noSpaces
// {"i":100,"what_am_i":"Foo"}

decode[Event]("""{ "i": 1000, "what_am_i": "Foo" }""")
// Right(Foo(i = 1000))
```

**Caveat:** Discriminator field can collide with case class member names. Not the default behavior for this reason.

### Shapeless Coproduct approach
```scala
import io.circe.shapes._
import shapeless.{ Coproduct, Generic }

implicit def encodeAdtNoDiscr[A, Repr <: Coproduct](implicit
  gen: Generic.Aux[A, Repr],
  encodeRepr: Encoder[Repr]
): Encoder[A] = encodeRepr.contramap(gen.to)

implicit def decodeAdtNoDiscr[A, Repr <: Coproduct](implicit
  gen: Generic.Aux[A, Repr],
  decodeRepr: Decoder[Repr]
): Decoder[A] = decodeRepr.map(gen.from)
```

Tries constructors in alphabetical order. Can be problematic with ambiguous case classes.

---

## Recursive ADTs

### Problem
Naive codec derivation for recursive types causes `StackOverflowError` or diverging implicits.

### Solution: Decoder.recursive (recommended)
```scala
import io.circe.{Json, Decoder, HCursor}
import io.circe.syntax._
import cats.syntax.all._

sealed trait Tree[A]
case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
case class Leaf[A](value: A) extends Tree[A]

implicit def branchDecoder[A](implicit DTA: Decoder[Tree[A]]): Decoder[Branch[A]] =
  Decoder.forProduct2[Branch[A], Tree[A], Tree[A]]("l", "r")(Branch.apply)

implicit def leafDecoder[A: Decoder]: Decoder[Leaf[A]] =
  Decoder[A].at("v").map(Leaf(_))

implicit def treeDecoder[A: Decoder]: Decoder[Tree[A]] =
  Decoder.recursive { implicit recurse =>
    List[Decoder[Tree[A]]](
      Decoder[Branch[A]].widen,
      Decoder[Leaf[A]].widen
    ).reduce(_ or _)
  }
```

**Key points:**
- `Decoder.recursive` uses the `Defer` type class to provide lazy access to the eventual instance.
- Recursive calls are **not tail recursive** and thus not fully stack safe.
- Safe for data structures with depth at most `log(size)` (e.g., balanced trees).

### Self-referencing alternative (manual caching)
```scala
implicit def treeDecoder[A: Decoder]: Decoder[Tree[A]] =
  new Decoder[Tree[A]] {
    private implicit val self: Decoder[Tree[A]] = this
    private val delegate =
      List[Decoder[Tree[A]]](
        Decoder[Branch[A]].widen,
        Decoder[Leaf[A]].widen
      ).reduce(_ or _)

    override def apply(c: HCursor): Decoder.Result[Tree[A]] = delegate(c)
  }
```

---

## Optics

Requires `circe-optics` dependency. Uses Monocle under the hood.

```scala
import io.circe._, io.circe.parser._
import io.circe.optics.JsonPath._
import io.circe.optics.JsonOptics._
import monocle.function.Plated

val json: Json = parse("""
{
  "order": {
    "customer": {
      "name": "Custy McCustomer",
      "contactDetails": {
        "address": "1 Fake Street, London, England",
        "phone": "0123-456-789"
      }
    },
    "items": [{
      "id": 123,
      "description": "banana",
      "quantity": 1
    }, {
      "id": 456,
      "description": "apple",
      "quantity": 2
    }],
    "total": 123.45
  }
}
""").getOrElse(Json.Null)
```

### Traversing with optics
```scala
// Extract a single value
val _phoneNum = root.order.customer.contactDetails.phone.string
val phoneNum: Option[String] = _phoneNum.getOption(json)

// Extract multiple values
val items: List[Int] = root.order.items.each.quantity.int.getAll(json)
```

### Modifying with optics
```scala
val doubleQuantities: Json => Json =
  root.order.items.each.quantity.int.modify(_ * 2)

val modifiedJson = doubleQuantities(json)
```

### Recursive transformation
```scala
// Convert all numbers to strings throughout a JSON document
Plated.transform[Json] { j =>
  j.asNumber match {
    case Some(n) => Json.fromString(n.toString)
    case None    => j
  }
}(json)
```

**Key concepts:**
- Optics separate the description of a traversal from its execution.
- Uses Scala `Dynamic` for the fluent `root.order.customer...` syntax.
- `.getOption(json)` returns `Option[T]` for single values.
- `.getAll(json)` returns `List[T]` for multiple values (via `.each`).
- `.modify(f)` returns a `Json => Json` transformation function.
- `.set(value)` returns a `Json => Json` that replaces the focused value.

---

## Configuration and Extras

Requires `circe-generic-extras`.

### Snake case field names
```scala
import io.circe.generic.extras._, io.circe.syntax._

implicit val config: Configuration =
  Configuration.default.withSnakeCaseMemberNames

@ConfiguredJsonCodec case class User(firstName: String, lastName: String)

User("John", "Doe").asJson
// {"first_name": "John", "last_name": "Doe"}
```

### Custom field name transformation
```scala
implicit val config: Configuration = Configuration.default.copy(
  transformMemberNames = {
    case "i" => "my-int"
    case other => other
  }
)

@ConfiguredJsonCodec case class Bar(i: Int, s: String)
// {"my-int": 42, "s": "hello"}
```

### @JsonKey annotation (per-field override)
```scala
implicit val config: Configuration = Configuration.default

@ConfiguredJsonCodec case class Bar(@JsonKey("my-int") i: Int, s: String)
// {"my-int": 42, "s": "hello"}
```

### Constructor name transformation (ADTs)
```scala
implicit val config: Configuration = Configuration.default.copy(
  transformConstructorNames = _.toLowerCase
)

sealed trait Animal
case class Dog(age: Int) extends Animal
case class Cat(color: String) extends Animal

// Discriminator values become "dog", "cat" instead of "Dog", "Cat"
```

### Default values
```scala
implicit val config: Configuration = Configuration.default.copy(
  useDefaults = true
)

case class User(firstName: String, lastName: String = "Doe")

decode[User]("""{"firstName": "Foo"}""")
// Right(User(firstName = "Foo", lastName = "Doe"))
```

### Strict decoding (reject unknown fields)
```scala
implicit val config: Configuration = Configuration.default.copy(
  strictDecoding = true
)

case class User(firstName: String, lastName: String)

decode[User]("""{"firstName": "Foo", "lastName": "Bar", "likesCats": true}""")
// Left(DecodingFailure - Unexpected field: [likesCats])
```

### Configuration options summary

| Option | Default | Purpose |
|---|---|---|
| `transformMemberNames` | identity | Transform case class field names to JSON keys |
| `transformConstructorNames` | identity | Transform sealed trait subclass names (for ADT discriminators) |
| `useDefaults` | `false` | Use case class default values when fields are missing |
| `discriminator` | `None` | Field name for ADT type discriminator (set via `withDiscriminator`) |
| `strictDecoding` | `false` | Reject JSON with unexpected fields |

---

## Common Pitfalls and FAQ

| Pitfall | Fix |
|---|---|
| Using auto derivation in production | Use semi-auto (`deriveEncoder`/`deriveDecoder`) for explicit control and faster compiles |
| Missing `import io.circe.syntax._` | Required for `.asJson` and `.as[T]` extension methods |
| `Long` precision loss on Scala.js | `JSON.parse` converts to JS numbers; represent large numbers as strings |
| ADT decoder order matters | Decoders in `reduceLeft(_ or _)` are tried left to right; put most specific first |
| Forgetting `.widen` on ADT decoders | Circe type classes are invariant; `.widen` upcasts `Decoder[SubType]` to `Decoder[SuperType]` |
| Recursive ADT stack overflow | Use `Decoder.recursive` instead of naive implicit resolution |
| `@JsonCodec` not working | Need `-Ymacro-annotations` compiler flag (Scala 2.13+) |
| `-Ypartial-unification` with auto derivation | Incomplete decoders won't work (circe #724) |
| Old Scala (< 2.12) flatMap errors | Add `import cats.syntax.either._` |
| Extra fields decoded silently | Default behavior; use `Configuration.default.copy(strictDecoding = true)` to reject |

**FAQ**

**Q: When should I use `forProductN` vs `deriveEncoder`/`deriveDecoder`?**
A: Use `forProductN` when you need custom JSON field names different from case class field names and want to avoid the `circe-generic` / Shapeless dependency. It only requires `circe-core`.

**Q: How do I handle optional fields?**
A: `Option[A]` fields are automatically handled. Missing fields decode to `None`. Present fields decode the inner value. On encoding, `None` fields are omitted by default.

**Q: How do I pick between cursor traversal and optics?**
A: Use cursors (`HCursor`) for simple, one-off extractions. Use optics (`JsonPath`) for complex, reusable traversals and modifications, especially when you need to transform deeply nested structures.

**Q: How do I handle a JSON field that can be multiple types?**
A: Use `Decoder.instance` with manual cursor navigation, or combine decoders with `.or`: `Decoder[Int].map(_.toString) or Decoder[String]`.

---

## Test Prompts

1. "Define a case class hierarchy for a REST API response with nested objects and derive semi-auto codecs for it."
2. "Parse a JSON string, extract a deeply nested field using cursor navigation, modify it, and print the result."
3. "Create custom Encoder/Decoder for a sealed trait ADT with a type discriminator field."
4. "Implement codecs for a case class where JSON field names use snake_case but Scala fields use camelCase."
5. "Write an accumulating decoder that collects all validation errors for a complex JSON payload."
6. "Use circe optics to extract all prices from a nested JSON array of products and double them."
7. "Implement a recursive ADT (expression tree) with proper circe codecs that avoid stack overflow."
