## 暗黙値解決のデバッグ {#sec:generic:debugging}

```tut:book:invisible
import shapeless._

trait CsvEncoder[A] {
  val width: Int
  def encode(value: A): List[String]
}

object CsvEncoder {
  def apply[A](implicit enc: CsvEncoder[A]): CsvEncoder[A] =
    enc
}

def createEncoder[A](w: Int)(func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    val width = w
    def encode(value: A): List[String] =
      func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(1)(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(1)(num => List(num.toString))

implicit val doubleEncoder: CsvEncoder[Double] =
  createEncoder(1)(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(1)(bool => List(if(bool) "yes" else "no"))

implicit def optionEncoder[A](implicit encoder: CsvEncoder[A]): CsvEncoder[Option[A]] =
  createEncoder(encoder.width)(opt => opt.map(encoder.encode).getOrElse(List.fill(encoder.width)("")))

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(0)(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] =
  createEncoder(hEncoder.value.width + tEncoder.width) {
    case h :: t =>
      hEncoder.value.encode(h) ++ tEncoder.encode(t)
  }

implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(0)(cnil => ???)

implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: Lazy[CsvEncoder[H]],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] =
  createEncoder(hEncoder.value.width + tEncoder.width) {
    case Inl(h) => hEncoder.value.encode(h) ++ List.fill(tEncoder.width)("")
    case Inr(t) => List.fill(hEncoder.value.width)("") ++ tEncoder.encode(t)
  }

implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  enc: Lazy[CsvEncoder[R]]
): CsvEncoder[A] =
  createEncoder(enc.value.width)(a => enc.value.encode(gen.to(a)))
```

暗黙値の解決における不具合はややこしく、苛立たしいものになりうる。
暗黙値解決がうまくいかないときに使える、いくつかのテクニックを挙げる。

### *implicitly* を用いたデバッグ

単純に、コンパイラが暗黙の値を見つけるのに失敗している場合はどうすればよいのだろうか?
この問題は、使われている暗黙値のうちのどれかの解決が原因となっていることがある。
例えば:

```tut:book:silent
case class Foo(bar: Int, baz: Float)
```

```tut:book:fail
CsvEncoder[Foo]
```

この失敗の理由は、`Float` に対する `CsvEncoder` をまだ定義していないことにある。
しかし、アプリケーションコードの中ではこの原因が明白ではないこともある。
エラーの原因を発見するために、期待する型の展開列の各段階について、`CsvEncoder.apply` や `implicitly` の呼び出しをエラー箇所の上に挿入してコンパイルが通るかどうかを確認することができる。
まず、`Foo` のジェネリック表現から始める:

```tut:book:fail
CsvEncoder[Int :: Float :: HNil]
```

これは失敗するので、展開列のより深いところを探す必要があるとわかる。
次のステップでは、`HList` の各要素を試す:

```tut:book:silent
CsvEncoder[Int]
```

```tut:book:fail
CsvEncoder[Float]
```

`Int` では通るが、`Float` では失敗する。
`CsvEncoder[Float]` は型の「展開木」の葉なので、まずはこの不足しているインスタンスを実装すればいいとわかる。
インスタンスを追加しても問題が解決しない場合は、これを繰り返して次の失敗箇所を探すことになる。

### *reify* を用いたデバッグ

`scala.reflect` にある `reify` メソッドは、Scala の式を引数にとり、その式を表現する、完全な型アノテーション付きの抽象構文木のオブジェクトを返す:

```tut:book:silent
import scala.reflect.runtime.universe._
```

```tut:book
println(reify(CsvEncoder[Int]))
```

暗黙値の解決の途中で推論された型が、問題解決の手がかりとなることがある。
暗黙値の解決のあとに、`A` や `T` のような存在型(existential type) が残っている場合、これは何かがうまくいっていない印である。
同様に、`Any` や `Nothing` のような「トップ」や「ボトム」型も、問題の所在の証拠となる。
