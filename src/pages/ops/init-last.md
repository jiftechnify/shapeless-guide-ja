## シンプルな操作の例

`HList` は、`init` と `last` という拡張メソッドを持つ。これらはそれぞれ `shapeless.ops.hlist.Init` と `shapeless.ops.hlist.Last` という2つの型クラスに基づいている。
`init` は `HList` の最後の要素を削除し、一方 `last` は最後の要素を除くすべての要素を削除する。
`Coproduct` も同様のメソッドと型クラスを持つ。
これらは演算(ops)パターンの完璧な実例となる。
これらの拡張メソッドの単純化した定義は以下のようなものである:

```scala
package shapeless
package syntax

implicit class HListOps[L <: HList](l: L) {
  def last(implicit last: Last[L]): last.Out = last.apply(l)
  def init(implicit init: Init[L]): init.Out = init.apply(l)
}
```

それぞれのメソッドの返り値の型は、暗黙の引数の依存型によって決まる。
各型クラスに対するインスタンスが、実際の変換を提供している。
例として、`Last` の定義の骨格を以下に示す:

```scala
trait Last[L <: HList] {
  type Out
  def apply(in: L): Out
}

object Last {
  type Aux[L <: HList, O] = Last[L] { type Out = O }
  implicit def pair[H]: Aux[H :: HNil, H] = ???
  implicit def list[H, T <: HList]
    (implicit last: Last[T]): Aux[H :: T, last.Out] = ???
}
```

この実装から、いくつかの興味深い洞察を得ることができる。
まず、多くの場合、演算型クラスに対して少数のインスタンスだけを定義することになる(この場合、たった2つだ)。
よって、**すべて** の必要なインスタンスを型クラスのコンパニオンオブジェクトにまとめることで、`shapeless.ops` から何もインポートすることなく対応する拡張メソッドを呼び出せるようにすることができる:

```tut:book:silent
import shapeless._
```

```tut:book
("Hello" :: 123 :: true :: HNil).last
("Hello" :: 123 :: true :: HNil).init
```

2つ目に、この型クラスは少なくとも1つの要素を持つような `HList` に対してのみ定義されている。
これにより、ある程度の静的な検査を行えるようになる。
空の `HList` に対して `last` を呼び出そうとすると、コンパイルエラーとなる:

```tut:book:fail
HNil.last
```
