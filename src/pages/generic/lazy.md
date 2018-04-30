## 再帰型に対する型クラスインスタンスの導出

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

もっと野心的なことに挑戦してみよう---二分木だ:

```tut:book:silent
sealed trait Tree[A]
case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
case class Leaf[A](value: A) extends Tree[A]
```

理論的には、この定義に対する CSV ライターを召喚するために必要な定義は揃っているはずだ。
しかし、`writeCsv` の呼び出しはコンパイルに失敗する:

```tut:book:fail
CsvEncoder[Tree[Int]]
```

問題は、この型が再帰的であることだ。
コンパイラは暗黙値を適用する無限ループを検知し、音を上げてしまう。

### 暗黙値の発散(divergence)

暗黙値の解決は探索のプロセスである。
コンパイラは、探索が解に「収束(converge)」しているかどうかを判断するためにヒューリスティクスを用いる。
このヒューリスティクスが、ある探索の分岐において満足な結果を生み出さなかった場合、コンパイラはこの分岐は収束しないと仮定して別の分岐へ移動する。

あるヒューリスティクスは特に無限ループを回避するために設計されている。
コンパイラが、ある探索の分岐において複数回同じ対象の型を見つけた場合、そこで諦めて別の分岐へ移動する。
`CsvEncoder[Tree[Int]]` の展開を見てみると、この状況が発生していることがわかる。
暗黙値の解決プロセスは次のように進む:

```scala
CsvEncoder[Tree[Int]]                          // 1
CsvEncoder[Branch[Int] :+: Leaf[Int] :+: CNil] // 2
CsvEncoder[Branch[Int]]                        // 3
CsvEncoder[Tree[Int] :: Tree[Int] :: HNil]     // 4
CsvEncoder[Tree[Int]]                          // 5 おっと…
```

1行目と5行目で `Tree[A]` を二度見たので、コンパイラは別の探索の分岐へ移動する。
最終的な結果としては、適切な暗黙値を見つけるのに失敗する。

実際には、状況はもっと悪い。
コンパイラが同じ型コンストラクタを複数回見つけ、さらに型パラメータの複雑さが **増大している** 場合、コンパイラはその探索の分岐が「発散」したとみなす。
Shapeless にとってこれは問題だ。なぜなら、`::[H, T]` や `:+:[H, T]` のような型はコンパイラが別々のジェネリック表現を展開する際に何度か現れうるからだ。
これにより、同じ展開を続けていれば見つけられたはずの解にたどり着く前にコンパイラは諦めてしまう。
次のような型を考えてみよう:

```tut:book:silent
case class Bar(baz: Int, qux: String)
case class Foo(bar: Bar)
```

`Foo` の展開は次のようになる:

```scala
CsvEncoder[Foo]                   // 1
CsvEncoder[Bar :: HNil]           // 2
CsvEncoder[Bar]                   // 3
CsvEncoder[Int :: String :: HNil] // 4 あらら…
```

この探索において、コンパイラは `CsvEncoder[::[H, T]]` という型を二度(2行目と4行目)解決しようと試みる。
4行目における型パラメータ `T` は2行目のものよりも複雑なので、コンパイラはこの探索の分岐が発散したとみなす(この場合、この仮定は誤りだ)。
コンパイラは別の分岐へ移り、やはり適切なインスタンスの生成に失敗する。

### *Lazy*

暗黙値の発散は、 shapeless のようなライブラリにとって致命的な問題である。
幸い、shapeless はこれに対処する `Lazy` と呼ばれる型を提供している。
`Lazy` は2つのことを行う:

 1. 前述のような防御的すぎる収束判定をガードすることで、コンパイル時の暗黙値の発散を抑制する

 2. 暗黙の引数の評価を実行時まで先送りすることで、自己参照的な暗黙値の導出を許す

`Lazy` は、特定の暗黙の引数を包んで利用する。
経験則から言えば、すべての`HList` や `Coproduct` の規則の「先頭(head)」の引数と、すべての `Generic` の `Repr` 引数を `Lazy` に包むとよい:

```tut:book:invisible:reset
// Forward definitions -------------------------
import shapeless._

trait CsvEncoder[A] {
  def encode(value: A): List[String]
}

object CsvEncoder {
  def apply[A](implicit enc: CsvEncoder[A]): CsvEncoder[A] =
    enc
}

def createEncoder[A](func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    def encode(value: A): List[String] =
      func(value)
  }

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(num => List(num.toString))

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(hnil => Nil)

implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(cnil => throw new Exception("Inconceivable!"))

// ----------------------------------------------
```

```tut:book:silent
implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: Lazy[CsvEncoder[H]], // Lazy で包む
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = createEncoder {
  case h :: t =>
    hEncoder.value.encode(h) ++ tEncoder.encode(t)
}
```

```tut:book:silent
implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: Lazy[CsvEncoder[H]], // Lazy で包む
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] = createEncoder {
  case Inl(h) => hEncoder.value.encode(h)
  case Inr(t) => tEncoder.encode(t)
}
```

```tut:book:silent
implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  rEncoder: Lazy[CsvEncoder[R]] // Lazy で包む
): CsvEncoder[A] = createEncoder { value =>
  rEncoder.value.encode(gen.to(value))
}
```

```tut:book:invisible
sealed trait Tree[A]
final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]
```

これにより、コンパイラが探索の途中で諦めることを防ぐことができ、`Tree` のような複雑・再帰的な型に対する解を得られるようになる:

```tut:book
CsvEncoder[Tree[Int]]
```
