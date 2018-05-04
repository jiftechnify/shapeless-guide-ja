## *Poly* を用いた型クラスの定義 {#sec:poly:product-mapper}

`Poly` と `Mapper`、`FlatMapper` のような型クラスを構成要素として、独自の型クラスを構成できる。
例として、あるケースクラスを他のケースクラスに変換するような型クラスを作ってみよう:

```tut:book:silent
trait ProductMapper[A, B, P] {
  def apply(a: A): B
}
```

`Mapper` と一組の `Generics` を用いて、`ProductMapper` のインスタンスを生成できる:

```tut:book:silent
import shapeless._
import shapeless.ops.hlist

implicit def genericProductMapper[
  A, B,
  P <: Poly,
  ARepr <: HList,
  BRepr <: HList
](
  implicit
  aGen: Generic.Aux[A, ARepr],
  bGen: Generic.Aux[B, BRepr],
  mapper: hlist.Mapper.Aux[P, ARepr, BRepr]
): ProductMapper[A, B, P] =
  new ProductMapper[A, B, P] {
    def apply(a: A): B =
      bGen.from(mapper.apply(aGen.to(a)))
  }
```

興味深いことに、`Poly` に対する `P`型を定義しているにもかかわらず、コード中に `P`型の値を参照している箇所はない。
`Mapper` 型クラスは `Case` を見つけるのに暗黙値の解決を利用しているので、コンパイラは関連するインスタンスを特定するための `P` というシングルトン型を知ることさえできれば十分なのだ。

`ProductMapper` を使いやすくするために、拡張メソッドを作成しよう。
呼び出し時、ユーザが `B` の型だけを指定すればいいようにするために、遠回りして、コンパイラが値パラメータの型から `Poly` の型を推論できるようにする:

```tut:book:silent
implicit class ProductMapperOps[A](a: A) {
  class Builder[B] {
    def apply[P <: Poly](poly: P)
        (implicit pm: ProductMapper[A, B, P]): B =
      pm.apply(a)
  }

  def mapTo[B]: Builder[B] = new Builder[B]
}
```

このメソッドの利用例を示す:

```tut:book:silent
object conversions extends Poly1 {
  implicit val intCase:  Case.Aux[Int, Boolean]   = at(_ > 0)
  implicit val boolCase: Case.Aux[Boolean, Int]   = at(if(_) 1 else 0)
  implicit val strCase:  Case.Aux[String, String] = at(identity)
}

case class IceCream1(name: String, numCherries: Int, inCone: Boolean)
case class IceCream2(name: String, hasCherries: Boolean, numCones: Int)
```

```tut:book
IceCream1("Sundae", 1, false).mapTp[IceCream2](conversions)
```

`mapTo` 構文は1つのメソッド呼び出しのように見えるが、実際には2つの呼び出しが発生している:
1つは型パラメータ `B` を固定するための `mapTo` の呼び出し、もう1つは `Poly` を指定するための `Builder.apply` の呼び出しだ。
Shapeless の組み込みの演算拡張メソッドは、ユーザに便利な構文を提供するためにこれと同様のトリックを利用している。
