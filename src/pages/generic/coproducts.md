## 余積型に対する型クラスインスタンスの導出 {#sec:generic:coproducts}

```tut:book:invisible
// ----------------------------------------------
// Forward definitions

trait CsvEncoder[A] {
  def encode(value: A): List[String]
}

object CsvEncoder {
  def apply[A](implicit enc: CsvEncoder[A]): CsvEncoder[A] =
    enc
}

def writeCsv[A](values: List[A])(implicit encoder: CsvEncoder[A]): String =
  values.map(encoder.encode).map(_.mkString(",")).mkString("\n")

def createEncoder[A](func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    def encode(value: A): List[String] =
      func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(bool => List(if(bool) "cone" else "glass"))

import shapeless.{HList, HNil, ::}

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = createEncoder {
  case h :: t =>
    hEncoder.encode(h) ++ tEncoder.encode(t)
}

import shapeless.Generic

implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  rEncoder: CsvEncoder[R]
): CsvEncoder[A] =
  createEncoder(value => rEncoder.encode(gen.to(value)))

// ----------------------------------------------
```

前節では、任意の積型に対する `CsvEncoder` を自動的に導出するための一連の規則を作り出した。
本節では、同じパターンを余積型に適用する。
図形(shape) の例に戻ろう:

```tut:book:silent
sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape
```

`Shape` のジェネリック表現は `Rectangle :+: Circle :+: CNil` だ。
[@sec:generic:product-generic]節では、`Rectangle` と `Circle` に対するエンコーダを定義した。
ここで、`:+:` と `CNil` に対するジェネリックな `CsvEncoder` を書くために、`HList` に対して用いたのと同じ原則を用いる:

```tut:book:silent
import shapeless.{Coproduct, :+:, CNil, Inl, Inr}

implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(cnil => throw new Exception("Inconceivable!"))

implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] = createEncoder {
  case Inl(h) => hEncoder.encode(h)
  case Inr(t) => tEncoder.encode(t)
}
```

注意すべき2つのポイントがある:

 1. `Coproduct` は型の **選言** なので、`:+:` のエンコーダは左と右のどちらの値をエンコードするかを **選択** する必要がある。
    そこで、`:+:` の2つのサブ型に対してパターンマッチを行う。`Inl` は左、`Inr`は右の値に対応する。

 2. なんと、`CNil` のエンコーダは例外を発生させている!
    でも、落ち着いてほしい。
    `CNil`型の値を生成することはできないということを思い出そう。そのため、`throw` 式はデッドコードとなる。
    この地点には決して到達しないので、ここで突然失敗しても問題はない。

これらの定義を[@sec:generic:products]節の積型のエンコーダと合わせることで、図形のリストをシリアライズすることができるはずだ。
さて、やってみよう:

```tut:book:silent
val shapes: List[Shape] = List(
  Rectangle(3.0, 4.0),
  Circle(1.0)
)
```

```tut:book:fail
writeCsv(shapes)
```

おっと、失敗してしまった!
先程述べたように、エラーメッセージは役に立たない。
この失敗の原因は、`Double` に対する `CsvEncoder` のインスタンスがないことだ:

```tut:book:silent
implicit val doubleEncoder: CsvEncoder[Double] =
  createEncoder(d => List(d.toString))
```

この定義があれば、すべてが思ったとおりに動く:

```tut:book
writeCsv(shapes)
```

<div class="callout callout-warning">
  **SI-7046 とあなた**

  [SI-7046][link-si7046]と呼ばれる Scala コンパイラのバグがあり、これによって余積型のジェネリック表現の解決に失敗することがある。
  このバグは、 shapeless が依存しているマクロ API のある部分が、ソースコードの定義の順序に影響されることによって起こる。
  コードを並び替えてファイル名を変更することでこの問題に対処できるが、このような対処は場当たり的で信頼できないものになりがちである。

  Lightbend Scalaのバージョン 2.11.8 以前を利用していて、余積型の解決に失敗する場合は、Lightbend Scala 2.11.9 または Typelevel Scala 2.11.8 にアップグレードすることを考えたほうがよい。
  これらのリリースでは SI-7046 は修正されている。
</div>

### CSV 出力の整形

我々の CSV エンコーダはこのままではあまり実用的ではない。
`Rectangle` と `Circle` のフィールドが同じ列に並んでしまう。
この問題を修正するには、データの幅を取得し、それに基づいて出力に適切なスペースを挿入するように、`CsvEncoder`の定義を変更する必要がある。
[@sec:intro:source-code]節のリンク先にある実例のリポジトリは、この問題に対処した `CsvEncoder` の完全な実装を含んでいる。
