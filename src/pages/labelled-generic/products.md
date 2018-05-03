## *LabelledGeneric* を用いた積型の型クラスインスタンスの導出

`LabelledGeneric` について説明するために、JSON のエンコードという実用的な例を用いることにする。
値を JSON の抽象構文木に変換する、`JsonEncoder` という型クラスを定義する。
この手法は、Argonaut, Circe, Play JSON, Spray JSON, その他の Scala における JSON ライブラリで用いられているものだ。

まず、JSON データ型を定義する:

```tut:book:silent
sealed trait JsonValue
case class JsonObject(fields: List[(String, JsonValue)]) extends JsonValue
case class JsonArray(items: List[JsonValue]) extends JsonValue
case class JsonString(value: String) extends JsonValue
case class JsonNumber(value: Double) extends JsonValue
case class JsonBoolean(value: Boolean) extends JsonValue
case object JsonNull extends JsonValue
```

次に、値を JSON にエンコードするための型クラスを定義する:

```tut:book:silent
trait JsonEncoder[A] {
  def encode(value: A): JsonValue
}

object JsonEncoder {
  def apply[A](implicit enc: JsonEncoder[A]): JsonEncoder[A] = enc
}
```

さらに、いくつかの基本のインスタンスを定義する:

```tut:book:silent
def createEncoder[A](func: A => JsonValue): JsonEncoder[A] =
  new JsonEncoder[A] {
    def encode(value: A): JsonValue = func(value)
  }

implicit val stringEncoder: JsonEncoder[String] =
  createEncoder(str => JsonString(str))

implicit val doubleEncoder: JsonEncoder[Double] =
  createEncoder(num => JsonNumber(num))

implicit val intEncoder: JsonEncoder[Int] =
  createEncoder(num => JsonNumber(num))

implicit val booleanEncoder: JsonEncoder[Boolean] =
  createEncoder(bool => JsonBoolean(bool))
```

そして、いくつかのインスタンスコンビネータを定義する:

```tut:book:silent
implicit def listEncoder[A]
    (implicit enc: JsonEncoder[A]): JsonEncoder[List[A]] =
  createEncoder(list => JsonArray(list.map(enc.encode)))

implicit def optionEncoder[A]
    (implicit enc: JsonEncoder[A]): JsonEncoder[Option[A]] =
  createEncoder(opt => opt.map(enc.encode).getOrElse(JsonNull))
```

理想的には、ADT を JSON に変換する際は、出力の JSON に適切なフィールド名を用いたいところだ:

```tut:book:silent
case class IceCream(name: String, numCherries: Int, inCone: Boolean)

val iceCream = IceCream("Sundae", 1, false)

// 理想的には、次のようなものを出力したい:
val iceCreamJson: JsonValue =
  JsonObject(List(
    "name"        -> JsonString("Sundae"),
    "numCherries" -> JsonNumber(1),
    "inCone"      -> JsonBoolean(false)
  ))
```

ここで `LabelledGeneric` の登場だ。
`IceCream` に対するインスタンスを召喚し、出力される表現がどんなものかを見てみよう:

```tut:book:silent
import shapeless.LabelledGeneric
```

```tut:book
val gen = LabelledGeneric[IceCream].to(iceCream)
```

この `HList` の完全な型を明示すると次のようになる:

```scala
// String  with KeyTag[Symbol with Tagged["name"], String]     ::
// Int     with KeyTag[Symbol with Tagged["numCherries"], Int] ::
// Boolean with KeyTag[Symbol with Tagged["inCone"], Boolean]  ::
// HNil
```

この型は、これまで見てきたものよりも少し複雑だ。
Shapeless は、フィールド名をリテラル文字列型で表現するのではなく、リテラル文字列型でタグ付けされたシンボルによって表現する。
この実装の詳細は特段重要というわけではない:
`Witness` と `FieldType` を用いてタグを抽出することができるのは同じだが、`String` ではなく `Symbol` として現れているというだけのことだ[^future-tags]。

[^future-tags]: shapeless の将来のバージョンでは、`String` をタグとして用いるようになるかもしれない。

### *HList* に対するインスタンス

さて、`HNil` と `::` に対する  `JsonEncoder` のインスタンスを定義しよう。
このエンコーダは `JsonObject` を生成し、操作することになるので、これを簡単にするためにエンコーダの新しい型を導入する:

```tut:book:silent
trait JsonObjectEncoder[A] extends JsonEncoder[A] {
  def encode(value: A): JsonObject
}

def createObjectEncoder[A](fn: A => JsonObject): JsonObjectEncoder[A] =
  new JsonObjectEncoder[A] {
    def encode(value: A): JsonObject =
      fn(value)
  }
```

これで、`HNil` に対する定義は自明なものとなる:

```tut:book:silent
import shapeless.{HList, ::, HNil, Lazy}

implicit val hnilEncoder: JsonObjectEncoder[HNil] =
  createObjectEncoder(hnil => JsonObject(Nil))
```

`hkistEncoder` の定義はいくつかの要素を含むことになるので、ひとつひとつ見ていこう。
まず始めに、通常の `Generic` を用いた場合の定義がどのようなものになるかを考えてみる:

```tut:book:silent
implicit def hlistObjectEncoder[H, T <: HList](
  implicit
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonEncoder[H :: T] = ???
```

`LabelledGeneric` はタグ付けされた型の `HList` をもたらすので、まずキーの型に対応する新しい型変数を導入しよう:

```tut:book:silent
import shapeless.Witness
import shapeless.labelled.FieldType

implicit def hlistObjectEncoder[K, H, T <: HList](
  import
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonObjectEncoder[FieldType[K, H] :: T] = ???
```

メソッドの本体では、`K` に対応付けられた値が必要となる。
そのために、暗黙の `Witness` 引数を追加する:

```tut:book:silent
implicit def hlistObjectEncoder[K, H, T <: HList](
  implicit
  witness: Witness.Aux[K],
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonObjectEncoder[FieldType[K, H] :: T] = {
  val fieldName = witness.value
  ???
}
```

`witness.value` を用いて `K` の値にアクセスすることはできるが、我々が取得しようとしているタグの型について、コンパイラに知る術はない。
`LabelledGeneric` は タグとして `Symbol` を用いているので、`K` に型境界を設け、`symbol.name` によってタグを `String` に変換する:

```tut:book:silent
implicit def hlistObjectEncoder[K <: Symbol, H, T <: HList](
  implicit
  witness: Witness.Aux[K],
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonObjectEncoder[FieldType[K, H] :: T] = {
  val fieldName: String = witness.value.name
  ???
}
```

残りの定義では、[@sec:generic]章で説明した原則を用いる:

```tut:book:silent
implicit def hlistObjectEncoder[K <: Symbol, H, T <: HList](
  implicit
  witness: Witness.Aux[K],
  hEncoder: Lazy[JsonEncoder[H]],
  tEncoder: JsonObjectEncoder[T]
): JsonObjectEncoder[FieldType[K, H] :: T] = {
  val fieldName: String = witness.value.name
  createObjectEncoder { hList =>
    val head = hEncoder.value.encode(hlist.head)
    val tait = tEncoder.encode(hlist.tail)
    JsonObject((fieldName, head) :: tail.fields)
  }
}
```

### 具体的な積型に対するインスタンス

最後に、ジェネリックなインスタンスの定義に戻ろう。
これは、`Generic` のかわりに `LabelledGeneric` を利用していることを除けば、前に見た定義と同じものだ:

```tut:book:silent
import shapeless.LabelledGeneric

implicit def genericObjectEncoder[A, H](
  implicit
  generic: LabelledGeneric.Aux[A, H],
  hEncoder: Lazy[JsonObjectEncoder[H]]
): JsonEncoder[A] =
  createObjectEncoder { value =>
    hEncoder.value.encode(generic.to(value))
  }
```

必要なものはすべて揃った!
これらの定義があれば、任意のケースクラスのインスタンスを、フィールド名を保ちながら JSON にシリアライズすることができる:

```tut:book
JsonEncoder[IceCream].encode(iceCream)
```
