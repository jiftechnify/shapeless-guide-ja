## 積型のインスタンスの導出  {#sec:generic:products}

本節では shapeless を用いて積型(あるいはケースクラス)に対する型クラスのインスタンスを導出していく。
ここでは2つの直感的洞察を利用する:

1. `HList` の先頭(head)と先頭以外の残り(tail)に対する型クラスのインスタンスがあれば、`HList` 全体に対するインスタンスを導出できる。

2. ケースクラス `A` と `Generic[A]`、さらにそのジェネリック表現の表現型 `Repr` に対する型クラスインスタンスがあれば、これらを組み合わせて `A`型に対するインスタンスを生成できる。

`CsvEncoder` と `IceCream` を例にとると:

 - `IceCream` は `Repr`の型が `String :: Int :: Boolean :: HNil` のジェネリック表現を持つ。

 - `Repr` は `String`、`Int`、`Boolean`、そして `HNil` からなる。
  これらの型に対する `CsvEncoder` があれば、全体の型に対するエンコーダーを生成できる。

 - `Repr` に対する `CsvEncoder` を導出できれば、`IceCream` に対する `CsvEncoder` も手に入る。

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

case class IceCream(name: String, numCherries: Int, inCone: Boolean)

val iceCreams: List[IceCream] = List(
  IceCream("Sundae", 1, false),
  IceCream("Cornetto", 0, true),
  IceCream("Banana Split", 0, false)
)

case class Employee(name: String, number: Int, manager: Boolean)

val employees: List[Employee] = List(
  Employee("Bill", 1, true),
  Employee("Peter", 2, false),
  Employee("Milton", 3, false)
)
// ----------------------------------------------
```

### *HList* に対する型クラスインスタンス

まず、`CsvEncoder` のインスタンスのコンストラクタと、`String`、`Int`、`Boolean`に対する `CsvEncoder` を定義するところから始めよう:

```tut:book:silent
def createEncoder[A](func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    def encode(value: A): List[String] = func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(bool => List(if(bool) "yes" else "no"))
```

`HList` のエンコーダを作り出すために、これらの構成要素を組み合わせることができる。
以下のような、`HNil` と `::` のそれぞれに1つずつ、合わせて2つの規則を用いる:

```tut:book:silent
import shapeless.{HList, ::, HNil}

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] =
  createEncoder {
    case h :: t =>
      hEncoder.encode(h) ++ tEncoder.encode(t)
  }
```

これらの合わせて5つのインスタンスにより、`String`、`Int`、`Boolean` からなる任意の `HList` に対する `CsvEncoder` を召喚できるようになる:

```tut:book:silent
val reprEncoder: CsvEncoder[String :: Int :: Boolean :: HNil] =
  implicitly
```

```tut:book
reprEncoder.encode("abc" :: 123 :: true :: HNil)
```

### 具体的な積型に対する型クラスインスタンス {#sec:generic:product-generic}

`HList` に対する導出規則を `Generic` のインスタンスと組み合わせることで、`IceCream` に対する `CsvEncoder` を作り出すことができる:

```tut:book:silent
import shapeless.Generic

implicit val iceCreamEncoder: CsvEncoder[IceCream] = {
  val gen = Generic[IceCream]
  val enc = CsvEncoder[gen.Repr]
  createEncoder(iceCream => enc.encode(gen.to(iceCream)))
}
```

そして、次のようにして利用できる:

```tut:book
writeCsv(iceCreams)
```

この解決策は `IceCream` に特有のものになっている。
理想的には、`Generic` とそれに対応する `CsvEncoder` を持つすべてのケースクラスを扱える1つの規則が欲しいところだ。
段階を踏んで、この導出を完成させていこう。
これが最初のコードだ:

```scala
implicit def genericEncoder[A](
  implicit
  gen: Generic[A],
  enc: CsvEncoder[???]
): CsvEncoder[A] = createEncoder(a => enc.encode(gen.to(a)))
```

最初に直面する問題は、`???` の場所に入れる型を選ぶことだ。
`gen` に対応付いた `Repr`型を書きたいところだが、それはできない:

```tut:book:fail
implicit def genericEncoder[A](
  implicit
  gen: Generic[A],
  enc: CsvEncoder[gen.Repr]
): CsvEncoder[A] =
  createEncoder(a => enc.encode(gen.to(a)))
```

ここでの問題は、スコープの問題だ:
同じブロックの中のある引数から、他の引数の型メンバーを参照することはできないのだ。
これを解決するトリックは、メソッドに新しい型パラメータを導入し、関連する値パラメータのそれぞれが、その型パラメータを参照するようにするというものだ:

```tut:book:silent
implicit def genericEncoder[A, R](
  implicit
  gen: Generic[A] { type Repr = R },
  enc: CsvEncoder[R]
): CsvEncoder[A] =
  createEncoder(a => enc.encode(gen.to(a)))
```

このコーディングスタイルについては、次章でより詳しく説明する。
今のところは、この定義はコンパイルを通って期待通りに動き、これを任意のケースクラスに対して利用できる、とだけ言っておこう。
直感的にいえば、この定義は次のようなことを表す:

> ある型 `A` と、`R`という型を持つ `HList`、
> `A` を `R` に対応付ける暗黙の `Generic`、
> そして `R` に対する `CsvEncoder` があるとき、
> `A` に対する `CsvEncoder` を生成する。

今や任意のケースクラスを扱える完全なシステムが手に入った。
コンパイラは次のような呼び出しを:

```tut:book:silent
writeCsv(iceCreams)
```

我々が定義した一連の導出規則を用いて展開する:

```tut:book:silent
writeCsv(iceCreams)(
  genericEncoder(
    Generic[IceCream],
    hlistEncoder(stringEncoder,
      hlistEncoder(intEncoder,
        hlistEncoder(booleanEncoder, hnilEncoder)))))
```

さらに、コンパイラは多くの様々な積型に対して正しい展開形を推論することができる。
このコードを自分の手で書かなくてもよい、というのは素敵なことだ! きっとそう思ってくれただろう。

<div class="callout callout-info">
**Aux 型エイリアス**

`Generic[A] {type Repr = L }` のような型の精密化(type refinement)は冗長で読みづらいので、shapeless は型メンバーを型パラメータとして言い換えるための  `Generic.Aux` という型エイリアスを提供している:

```scala
package shapeless

object Generic {
  type Aux[A, R] = Generic[A] { type Repr = R }
}
```

この別名を用いて、より読みやすい定義を書くことができる:

```tut:book:silent
implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  env: CsvEncoder[R]
): CsvEncoder[A] =
  createEncoder(a => env.encode(gen.to(a)))
```

`Aux` 型はセマンティクスを全く変えていないということに注意してほしい---ただコードをより読みやすくしているだけなのだ。
この"`Aux` パターン"は shapeless のコードベースで頻繁に用いられている。
</div>

### ならば、欠点はないのか?

ここまで見てきたすべてが、素晴らしい魔法のようだと思われたかもしれない。だが、ここでひとつの重要な「厳しい現実」を知らせなければならない。どうかお許し願いたい。
物事がうまく行かなかった場合、コンパイラはその理由をうまく伝えられないのだ。

上記のコードがコンパイルに失敗する場合、大きく分けて2つの理由がある。
1つ目はコンパイラが `Generic` のインスタンスを見つけられなかった場合だ。
例えば、ケースクラスでないクラスを引数として、`writeCsv` を呼び出してみよう:

```tut:book:silent
class Foo(bar: String, baz: Int)
```

```tut:book:fail
writeCsv(List(new Foo("abc", 123)))
```

この場合、エラーメッセージは比較的理解しやすい。
shapeless が `Generic` を計算できなかった場合、それは問題の型が ADT ではないということを意味する---代数のどこかに、ケースクラスや sealed 抽象型でない型があるということだ。

もうひとつの失敗の可能性の原因は、コンパイラが `HList` に対する  `CsvEncoder` を計算できないことだ。
これは通常、ADT の中のフィールドのひとつに対するエンコーダが存在しないために起こる。
例えば、我々はまだ `java.util.Date` に対する `CsvEncoder` を定義していないので、以下のコードは失敗する:

```tut:book:silent
import java.util.Date

case class Booking(room: String, date: Date)
```

```tut:book:fail
writeCsv(List(Booking("Lecture hall", new Date())))
```

ここで表示されるメッセージはあまり有益ではない。
コンパイラは、暗黙値の多くの組み合わせを試した結果うまくいかなかった、ということしか分からないのだ。
コンパイラは、どの組み合わせが望ましい結果に最も近いのかに関しては何もわからないので、失敗の原因がどこにあるのかを伝えることはできない。

さらに、あまりよくない知らせがある。
エラーの原因は、除去の過程を通して、自分の手で見つけ出さねばならないのだ。
[@sec:generic:debugging]節では、デバッグのテクニックについて説明する。
暗黙値の解決は必ずコンパイル時に失敗するということは、不幸中の幸いだ。
コードが実行中に失敗する見込みはほとんどない。
