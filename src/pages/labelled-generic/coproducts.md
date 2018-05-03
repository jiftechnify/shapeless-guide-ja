## *LabelledGeneric* を用いた余積型の型クラスインスタンスの導出

```tut:book:invisible:reset
// ----------------------------------------------
// Forward definitions:

sealed trait JsonValue
final case class JsonObject(fields: List[(String, JsonValue)]) extends JsonValue
final case class JsonArray(items: List[JsonValue]) extends JsonValue
final case class JsonString(value: String) extends JsonValue
final case class JsonNumber(value: Double) extends JsonValue
final case class JsonBoolean(value: Boolean) extends JsonValue
case object JsonNull extends JsonValue

trait JsonEncoder[A] {
  def encode(value: A): JsonValue
}

object JsonEncoder {
  def apply[A](implicit enc: JsonEncoder[A]): JsonEncoder[A] =
    enc
}

def createEncoder[A](func: A => JsonValue): JsonEncoder[A] =
  new JsonEncoder[A] {
    def encode(value: A): JsonValue =
      func(value)
  }

implicit val stringEncoder: JsonEncoder[String] =
  createEncoder(str => JsonString(str))

implicit val doubleEncoder: JsonEncoder[Double] =
  createEncoder(num => JsonNumber(num))

implicit val intEncoder: JsonEncoder[Int] =
  createEncoder(num => JsonNumber(num))

implicit val booleanEncoder: JsonEncoder[Boolean] =
  createEncoder(bool => JsonBoolean(bool))

implicit def listEncoder[A](implicit encoder: JsonEncoder[A]): JsonEncoder[List[A]] =
  createEncoder(list => JsonArray(list.map(encoder.encode)))

implicit def optionEncoder[A](implicit encoder: JsonEncoder[A]): JsonEncoder[Option[A]] =
  createEncoder(opt => opt.map(encoder.encode).getOrElse(JsonNull))

trait JsonObjectEncoder[A] extends JsonEncoder[A] {
  def encode(value: A): JsonObject
}

def createObjectEncoder[A](func: A => JsonObject): JsonObjectEncoder[A] =
  new JsonObjectEncoder[A] {
    def encode(value: A): JsonObject =
      func(value)
  }

import shapeless.{HList, ::, HNil, Witness, Lazy}
import shapeless.labelled.FieldType

implicit val hnilObjectEncoder: JsonObjectEncoder[HNil] =
  createObjectEncoder(hnil => JsonObject(Nil))

implicit def hlistObjectEncoder[K <: Symbol, H, T <: HList](
  implicit
  witness: Witness.Aux[K],
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonObjectEncoder[FieldType[K, H] :: T] = {
  val fieldName: String = witness.value.name
  createObjectEncoder { hlist =>
    val head = hEncoder.value.encode(hlist.head)
    val tail = tEncoder.encode(hlist.tail)
    JsonObject((fieldName, head) :: tail.fields)
  }
}

import shapeless.LabelledGeneric

implicit def genericObjectEncoder[A, H](
  implicit
  generic: LabelledGeneric.Aux[A, H],
  hEncoder: Lazy[JsonObjectEncoder[H]]
): JsonObjectEncoder[A] =
  createObjectEncoder(value => hEncoder.value.encode(generic.to(value)))

sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape

// ----------------------------------------------
```

`Coproduct` に `LabelledGeneric` を適用するには、既に説明した概念を組み合わせることになる。
まず始めに、`LabelledGeneric` によって導出される `Coproduct` の型を調べていこう。
[@sec:generic]章の `Shape` ADT を再び見てみよう:

```tut:book:silent
import shapeless.LabelledGeneric

sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape
```

```tut:book
LabelledGeneric[Shape].to(Circle(1.0))
```

`Coproduct` の型をより読みやすく整形すると、以下のようになる:

```scala
// Rectangle with KeyTag[Symbol with Tagged["Rectangle"], Rectangle] :+:
// Circle    with KeyTag[Symbol with Tagged["Circle"],    Circle]    :+:
// CNil
```

ご覧の通り、結果はそれぞれ型の名前でタグ付けされた、`Shape` のサブ型からなる `Coproduct` となる。
`:+:` と `CNil` に対する `JsonEncoder` を書く際に、この情報を利用できる:

```tut:book:silent
import shapeless.{Coproduct, :+:, CNil, Inl, Inr, Witness, Lazy}
import shapeless.labelled.FieldType

implicit val cnilObjectEncoder: JsonObjectEncoder[CNil] =
  createObjectEncoder(cnil => throw new Exception("Inconceivable!"))

implicit val coproductObjectEncoder[K <: Symbol, H, T <: Coproduct](
  implicit
  witness: Witness.Aux[K],
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonObjectEncoder[FieldType[K, H] :+: T] = {
  val typeName = witness.value.name
  createObjectEncoder {
    case Inl(h) =>
      JsonObject(List(typeName -> hEncoder.value.encode(h)))

    case Inr(t) =>
      tEncoder.encode(t)
  }
}
```

`coproductEncoder` は `hlistEncoder` と同様のパターンに従っている。
ここには3つの型パラメータがある:
型の名前を表す `K`、`Coproduct` の先頭の値の型 `H`、そして先頭以外の値の型 `T`だ。
結果の型ではこれらの3つの関係を示すために `FieldType` と `:+:` を用い、型名に実行時にアクセスするために `Witness` を利用している。
結果は、1つのキー・値ペアを含む。キーは型の名前で、値は中身の変換結果となる:

```tut:book:silent
val shape: Shape = Circle(1.0)
```

```tut:book
JsonEncoder[Shape].encode(shape)
```

少し工夫すれば、他の形式でエンコードすることもできる。
例えば、出力に `"type"` フィールドを追加したり、ユーザがフォーマットを設定できるようにすることさえできる。
Sam Halliday の [spray-json-shapeless][link-spray-json-shapeless] は、親しみやすく、それでいて多大な柔軟性を提供する、コードベースの優れた一例である。
