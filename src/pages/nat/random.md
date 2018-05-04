## 事例: ランダムな値の生成

[ScalaCheck][link-scalacheck] のようなプロパティベーステストライブラリは、ユニットテストのためのランダムなデータを生成するのに型クラスを利用している。
例えば、ScalaCheck は次のように利用できる `Arbitrary` 型クラスを提供している:

```tut:book:silent
import org.scalacheck._
```

```tut:book
for(i <- 1 to 3) println(Arbitrary.arbitrary[Int].sample)
for(i <- 1 to 3) println(Arbitrary.arbitrary[(Boolean, Byte)].sample)
```

ScalaCheck は、幅広い標準的な Scala の型に対する組み込みの `Arbitrary` のインスタンスを提供している。
しかし、ユーザ独自の ADT に対する `Arbitrary` のインスタンスを作成する仕事は、いまだに時間のかかる手動作業で行われている。
よって、[scalacheck-shapeless][link-scalacheck-shapeless] のようなライブラリを通した shapeless との統合は非常に魅力的なものだ。

本章では、ユーザ定義の ADT のランダム値を生成するための、シンプルな `Random` 型クラスを作成していく。
この実装において、`Length` や `Nat` が重要な役割を果たしていることを示す。
いつもの通り、まず型クラス自体の定義を行う:

```tut:book:invisible
// 常に同じ出力となるようにする
scala.util.Random.setSeed(0)
```

```tut:book:silent
trait Random[A] {
  def get: A
}

def random[A](implicit r: Random[A]): A = r.get
```

### 単純なランダム値

まず、いくつか基本的な `Random` のインスタンスを作る:

```tut:book:silent
// インスタンスのコンストラクタ:
def createRandom[A](func: () => A): Random[A] =
  new Random[A] {
    def get = func()
  }

// 0 から 9 までのランダムな数値
implicit val intRandom: Random[Int] =
  createRandom(() => scala.util.Random.nextInt(10))

// A から Z までのランダムな文字
implicit val charRandom: Random[Char] =
  createRandom(() => ('A'.toInt +
  scala.util.Random.nextInt(26)).toChar)

// ランダムなブール値
implicit val booleanRandom: Random[Boolean] =
  createRandom(() => scala.util.Random.nextBoolean)
```

次のようにして、`random` メソッドを通してこれらの単純なジェネレータを利用できる:

```tut:book
for(i <- 1 to 3) println(random[Int])
for(i <- 1 to 3) println(random[Char])
```

### ランダムな積型の値

[@sec:generic]章の `Generic` と `HList` を用いるテクニックによって、積型のランダム値を生成できる:

```tut:book:silent
import shapeless._

implicit def genericRandom[A, R](
  implicit
  gen: Generic.Aux[A, R],
  random: Lazy[Random[R]]
): Random[A] =
  createRandom(() => gen.from(random.value.get))

implicit val hnilRandom: Random[HNil] =
  createRandom(() => HNil)

implicit def hlistRandom[H, T <: HList](
  implicit
  hRandom: Lazy[Random[H]],
  tRandom: Random[T]
): Random[H :: T] =
  createRandom(() => hRandom.value.get :: tRandom.get)
```

これで、ケースクラスのランダム値を生成するところまではできるようになった:

```tut:book:silent
case class Cell(col: Char, row: Int)
```

```tut:book
for(i <- 1 to 5) println(random[Cell])
```

### ランダムな余積型の値

ここからが問題だ。
余積型のランダム値を生成するには、ランダムにサブ型を選び出す必要がある。
まず、素朴な実装を見てみよう:

```tut:book:silent
implicit val cnilRandom: Random[CNil] =
  createRandom(() => throw new Exception("Inconceivable!"))

implicit def coproductRandom[H, T <: Coproduct](
  implicit
  hRandom: Lazy[Random[H]],
  tRandom: Random[T]
): Random[H :+: T] =
  createRandom{ () =>
    val chooseH = scala.util.Random.nextDouble < 0.5
    if(chooseH) Inl(hRandom.value.get) else Inr(tRandom.get)
  }
```

この実装の問題は、`chooseH` の計算で半々の選択を行っているところにある。
これでは、確率分布にムラが出てしまう。
例えば、次のような型を考えてみよう:

```tut:book:silent
sealed trait Light
case object Red extends Light
case object Amber extends Light
case object Green extends Light
```

`Light` の `Repr` 型は `Red :+: Amber :+: Green :+: CNil` となる。
この型の `Random` のインスタンスは、`Red` と`Amber :+: Green :+: CNil` を それぞれ50%の確率で選択してしまう。
正しい配分は、`Red` を33%、`Amber :+: Green :+: CNil` を67%の確率で選ぶというものになるだろう。

問題はそれだけではない。
全体の確率分布を見ると、より危険な事態が発生することが分かる:

- `Red` は 1/2 の確率で選択される
- `Amber` は 1/4 の確率で選択される
- `Green` は 1/8 の確率で選択される
- **`CNil` が 1/16 の確率で選択される**

この余積型に対するインスタンスは、 6.75% の確率で例外を発生させるのだ!

```scala
for(i <- 1 to 100) random[Light]
// java.lang.Exception: Inconceivable!
//   ...
```

この問題を修正するには、`H` を選択する確率を変える必要がある。
正しい振る舞いは、`n` を余積型の長さとしたとき、 `H` を `1/n` の確率で選ぶというものになるだろう。
これにより、余積型のすべてのサブ型についてムラのない確率分布が保証される。
また、ただ1つのサブ型を持つ `Coproduct` が 100% の確率で選択されることも保証されるので、`cnilProduct.get` が呼び出されることは決してない。
これが新しい実装だ:

```tut:book:silent
import shapeless.ops.coproduct
import shapeless.ops.nat.ToInt

implicit def coproductRandom[H, T <: Coproduct, L <: Nat](
  implicit
  hRandom: Lazy[Random[H]],
  tRandom: Random[T],
  tLength: coproduct.Length.Aux[T, L],
  tLengthAsInt: ToInt[L]
): Random[H :+: T] = {
  createRandom { () =>
    val length = 1 + tLengthAsInt()
    val chooseH = scala.util.Random.nextDouble < (1.0 / length)
    if(chooseH) Inl(hRandom.value.get) else Inr(tRandom.get)
  }
}

```

この変更により、任意の積型や余積型のランダム値を生成できるようになった:

```tut:book
for(i <- 1 to 5) println(random[Light])
```

ScalaCheck のためのテストデータの生成は通常、大量のボイラープレートを必要とする。
ランダム値の生成は、`Nat` が核心的な構成要素となるような shapeless の強力なユースケースのひとつである。
