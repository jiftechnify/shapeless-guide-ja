## 依存型付けされた関数 {#sec:type-level-programming:depfun}

Shapeless はあらゆる場所で依存型を利用している。
具体的には、`Generic`、`Witness`(次章で詳しく見ていく)、そしてこの手引きの第2部で調べていく、他の多数の「演算」型クラスにおいて利用している。

例えば、shapeless は `HList` の最後の要素を返す `Last` と呼ばれる型クラスを提供している。
その簡略化された定義は以下のようなものとなっている:

```scala
package shapeless.ops.hlist

trait Last[L <: HList] {
  type Out
  def apply(in: L): Out
}
````

`Last` のインスタンスを召喚して、`HList` の中身を検査することができる。
以下の2つの例において、はじめの `HList` の型によって `Out` の型が変化することに注意してほしい:

```tut:book:silent
import shapeless.{HList, ::, HNil}

import shapeless.ops.hlist.Last
```

```tut:book
val last1 = Last[String :: Int :: HNil]
val last2 = Last[Int :: String :: HNil]
```

`Last` のインスタンスを一度召喚すれば、その `apply` メソッドを通してそれを値レベルで利用できる:

```tut:book
last1("foo" :: 123 :: HNil)
last2(321 :: "bar" :: HNil)
```

このコードは、2つの形でエラーから保護されている。
`Last` に対して定義された暗黙の値により、入力の  `HList` が少なくとも1つの要素を持つ場合にのみインスタンスを召喚できることが保証される:

```tut:book:fail
Last[HNil]
```

さらに、`Last` のインスタンスの型パラメータにより、期待されている型の `HList` を渡したかどうかチェックされる:

```tut:book:fail
last1(321 :: "bar" :: HNil)
```

より進んだ例として、`HList` の2番目の要素を返す `Second` という名前の独自の型クラスを定義してみよう:

```tut:book:silent
trait Second[L <: HList] {
  type Out
  def apply(value: L): Out
}

object Second {
  type Aux[L <: HList, O] = Second[L] { type Out = O }

  def apply[L <: HList](implicit inst: Second[L]): Aux[L, inst.Out] =
    inst
}
```

このコードでは、[@sec:generic:idiomatic-style]節で説明する慣習的なスタイルを用いている。
コンパニオンオブジェクトには、インスタンスを召喚するための標準の `apply` メソッドに加え、`Aux` 型を定義している。

<div class="callout callout-warning">
**召喚メソッド vs. "implicitly" vs. "the"**

`apply` の返り値型が `Second[L]` ではなく `Aux[L, O]` となっていることに注意しよう。
これは重要な違いだ。
`Aux` を用いることで、`apply` メソッドが召喚されたインスタンスの型メンバーを消去しないようにしている。
返り値の型を `Second[L]` と定義してしまうと、`Out` 型メンバーが返り値の方から消えて、型クラスが正しく動作しなくなってしまう。

`scala.Predef` の `implicit` メソッドはこのような動作をする。
`implicitly` によって召喚された `Last` のインスタンスの型は:

```tut:book
implicitly[Last[String :: Int :: HNil]]
```

これと、`Last.apply` によって召喚されたインスタンスの型を比べてみよう:

```tut:book
Last[String :: Int :: HNil]
```

`implicitly` によって召喚された型は `Out` 型メンバーを持っていない。
このような理由により、依存型付けされた関数を扱う際は `implicitly` の利用を避けるべきである。
独自の召喚メソッド、または shapeless が提供する代替メソッド `the` を利用しよう:

```tut:book:silent
import shapeless._
```

```tut:book
the[Last[String :: Int :: HNil]]
```
</div>

少なくとも2つの要素を持つ `HList` に対するインスタンスだけが必要となる:

```tut:book:silent
import Second._

implicit def hlistSecond[A, B, Rest <: HList]: Aux[A :: B :: Rest, B] =
  new Second[A :: B :: Rest] {
    type Out = B
    def apply(value: A :: B :: Rest): B =
      value.tail.head
  }
```

`Second.apply` によってインスタンスを召喚できる:

```tut:book:invisible
import Second._
```

```tut:book
val second1 = Second[String :: Boolean :: Int :: HNil]
val second2 = Second[String :: Int :: Boolean :: HNil]
```

この召喚は、`Last` と同様の制限を受ける。
互換性のない `HList` に対するインスタンスを召喚しようとすると、解決に失敗し、コンパイルエラーとなる:

```tut:book:fail
Second[String :: HNil]
```

召喚されたインスタンスは、適切な型の `HList` を値レベルで操作する `apply` メソッドを持つ:

```tut:book
second1("foo" :: true :: 123 :: HNil)
second2("bar" :: 321 :: false :: HNil)
```

```tut:book:fail
second1("baz" :: HNil)
```

## 依存型付け関数の連鎖 {#sec:type-level-programming:chaining}

依存型付けされた関数は、ある型から他の型を計算する手段を提供する。
複数のステップを含む計算を行うために、依存型付けされた関数を **連鎖** させることができる。
例えば、`Generic` を用いてケースクラスの `Repr` 型を計算し、`Last` を用いてそれの最後の要素の型を計算することができるはずだ。
これをコードにしてみよう:

```tut:book:invisible
import shapeless.Generic
```

```tut:book:fail
def lastField[A](input: A)(
  implicit
  gen: Generic[A],
  last: Last[gen.Repr]
): last.Out = last.apply(gen.to(input))
```

残念ながら、このコードはコンパイルを通らない。
これは[@sec:generic:product-generic]節における `genericEncoder` の定義と同じ問題を抱えている。
自由型変数を型パラメータに持ち上げることで、この問題に対処できる:

```tut:book:silent
def lastField[A, Repr <: HList](input: A)(
  implicit
  gen: Generic.Aux[A, Repr],
  last: Last[Repr]
): last.Out = last.apply(gen.to(input))
```

```tut:book:invisible
case class Vec(x: Int, y: Int)
case class Rect(origin: Vec, extent: Vec)
```

```tut:book
lastField(Rect(Vec(1, 2), Vec(3, 4)))
```

一般的な規則として、常にこのようなスタイルでコードを書くことになる。
すべての自由変数を型パラメータとして表し、コンパイラがそれを適切な型と照合できるようにする。
例えば、ちょうど1つのフィールドを持つケースクラスに対して `Generic` を召喚したいとしよう。
このように書きたくなるかもしれない:

```tut:book:silent
def getWrappedValue[A, H](input: A)(
  implicit
  gen: Generic.Aux[A, H :: HNil]
): H = gen.to(input).head
```

ここでの結果は、よりたちの悪いものだ。
このメソッド定義はコンパイルを通るが、コンパイラは呼び出し時に暗黙の値を決して見つけることができない:

```tut:book:silent
case class Wrapper(value: Int)
```

```tut:book:fail
getWrappedValue(Wrapper(42))
```

このエラーメッセージが問題のヒントを与えている。
`H` が出現していることが手がかりとなる。
これはメソッドの中の型パラメータの名前だ:
これはコンパイラが単一化を試みる型の中に現れるべきではないものである。
問題は、`gen` 引数が過剰に制限されていることだ:
コンパイラは、`Repr` の型とその長さを **同時に** 保証することはできないのだ。
`Nothing` という型も、問題の手がかりをもたらすことが多い。これはコンパイラが共変の型パラメータの単一化に失敗したときに現れる。

上記の問題の解決策は、暗黙値の解決を2段階に分割することだ:

1. `A` に対する適切な `Repr` を持つ `Generic` を見つける
2. 先頭の型が `H`  であるような `Repr` を見つける

`Repr` の型を制限するために `=:=` を用いて修正したメソッドは以下のようになる:

```tut:book:fail
def getWrappedValue[A, Repr <: HList, Head, Tail <: HList](input: A)(
  implicit
  gen: Generic.Aux[A, Repr],
  ev: (Head :: Tail) =:= Repr
): Head = gen.to(input).head
```

これはコンパイルを通らない。なぜなら、このメソッドの本体に含まれる `head` メソッド が `IsHCons` という方の暗黙の引数を要求するためだ。
これは比較的単純なエラーメッセージだ---ただ、shapeless の道具箱の中にあるツールについて学ぶことだけが必要だ。
`IsHCons` は、`HList` を `Head` と `Tail` に分割する shapeless の型クラスである。
`=:=` のかわりに `IsHCons` を用いることができる:

```tut:book:silent
import shapeless.ops.hlist.IsHCons

def getWrappedValue[A, Repr <: HList, Head](in: A)(
  implicit
  gen: Generic.Aux[A, Repr],
  isHCons: IsHCons.Aux[Repr, Head, HNil]
): Head = gen.to(in).head
```

これでバグが修正できた。
期待通り、メソッドの定義とその呼び出しの両方がコンパイルを通る:

```tut:book
getWrappedValue(Wrapper(42))
```

ここで覚えておいてほしいポイントは、`IsHCons` を使えば問題が解決するというところではない。
Shapeless はこれに似たたくさんの道具を提供しており([@sec:ops], [@sec:nat]章参照)、独自の型クラスの定義において、必要な場所でこれらを埋め合わせとして利用できる。
重要なのは、コンパイルを通り、解を見つけられるコードを書く過程を身につけることだ。
これまでの知見を要約したステップ・バイ・ステップのガイドをもって、本節の結びとする。
