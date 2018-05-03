## 独自の演算を作り出す(「定理」パターン) {#sec:ops:penultimate}

特定の操作の列が有用だと思ったら、それをまとめてもう1つの演算型クラスとして提供し直すことができる。
これは[@sec:type-level-programming:summary]節で紹介した「定理(lemma)」パターンの一例である。

演習として、独自の操作を作り出す課題に取り組んでいこう。
`Last` と `Init` の能力を組み合わせて、`HList` の最後から2番目の要素を取り出す `Penultimate` という型クラスを作っていく。
`Aux` 型エイリアスと `apply` メソッドを備えた型クラスの定義は次のようになる:

```tut:book:silent
import shapeless._

trait Penultimate[L] {
  type Out
  def apply(l: L): Out
}

object Penultimate {
  type Aux[L, O] = Penultimate[L] { type Out = O }

  def apply[L](implicit p: Penultimate[L]): Aux[L, p.Out] = p
}
```

やはり、`apply` メソッドの返り値の型が `Penultimate[L]` ではなく `Aux[L, O]` となっていることに注意してほしい。
[@sec:type-level-programming:depfun]節の別枠で説明したように、これによって召喚されたインスタンスの型メンバーが外から見えるようにしている。

[@sec:type-level-programming:chaining]節で説明したテクニックを利用して`Init` と `Last` を組み合わせ、`Penultimate` の1つのインスタンスを定義するだけでよい:

```tut:book:silent
import shapeless.ops.hlist

implicit def hlistPenultimate[L <: HList, M <: HList, O](
  implicit
  init: hlist.Init.Aux[L, M],
  last: hlist.Last.Aux[M, O]
): Penultimate.Aux[L, O] = {
  new Penultimate[L] {
    type Out = O
    def apply(l: L): O =
      last.apply(init.apply(l))
  }
}
```

次のように `Penultimate` を利用できる:

```tut:book:silent
type BigList = String :: Int :: Boolean :: Double :: HNil

val bigList: BigList = "foo" :: 123 :: true :: 456.0 :: HNil
```

```tut:book
Penultimate[BigList].apply(bigList)
```

`Penultimate` のインスタンスを召喚するには、コンパイラが `Last` と `Init` に対するインスタンスを召喚できなければならないので、短い `HList` に対する同水準の型検査を引き継ぐことができる:

```tut:book:silent
type TinyList = String :: HNil

val tinyList = "bar" :: HNil
```

```tut:book:fail
Penultimate[TinyList].apply(tinyList)
```

`HList` 上の拡張メソッドを定義すれば、利用するユーザがこれをより便利に使えるようにできる:

```tut:book:silent
implicit class PenultimateOps[A](a: A) {
  def penultimate(implicit inst: Penultimate[A]): inst.Out =
    inst.apply(a)
}
```

```tut:book
bigList.penultimate
```

`Generic` に基づくインスタンスを提供すれば、すべての積型に対して `Penultimate` を利用できるようになる:

```tut:book:silent
implicit def genericPenultimate[A, R, O](
  implicit
  generic: Generic.Aux[A, R],
  penultimate: Penultimate.Aux[R, O]
): Penultimate.Aux[A, O] =
  new Penultimate[A] {
    type Out = O
    def apply(a: A): O =
      penultimate.apply(generic.to(a))
  }

case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

```tut:book
IceCream("Sundae", 1, false).penultimate
```

ここで重要なことは、もう1つの型クラスとして `Penultimate` を定義することで、他の場所で適用できる再利用可能なツールを作り出したということだ。
Shapeless は多くの目的に応じたたくさんの演算を提供しているが、その道具箱に独自の演算を加えるのも容易なことなのだ。
