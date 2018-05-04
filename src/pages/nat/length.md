## ジェネリック表現の長さ

`Nat` の1つの使い道は、`HList` や `Coproduct` の長さを数えるというものだ。
Shapeless は、このために `shapeless.ops.hlist.Length` と `shapeless.ops.coproduct.Length` という型クラスを提供している:

```tut:book:silent
import shapeless._
import shapeless.ops.{hlist, coproduct, nat}
```

```tut:book
val hlistLength = hlist.Length[String :: Int :: Boolean :: HNil]
val coproductLength = coproduct.Length[Double :+: Char :+: CNil]
```

`Length` のインスタンスは、その長さを `Nat` の形で表現する型メンバー `Out` を持つ:

```tut:book
Nat.toInt[hlistLength.Out]
Nat.toInt[coproductLength.Out]
```

具体的な例でこれを利用してみよう。
ケースクラスのフィールドの数を数え、それを単純に `Int` として公開する `SizeOf` 型クラスを作成していく:

```tut:book:silent
trait SizeOf[A] {
  def value: Int
}

def sizeOf[A](implicit size: SizeOf[A]): Int = size.value
```

`SizeOf` のインスタンスを作るには、3つのものが必要となる:

1. 対応する `HList` 型を求めるための `Generic`
2. `HList` の長さを  `Nat` として求める `Length`
3. `Nat` を `Int` に変換する `ToInt`

[@sec:type-level-programming]章で説明したスタイルで書かれた、動作する実装は以下の通りだ:

```tut:book:silent
implicit def genericSizeOf[A, L <: HList, N <: Nat](
  implicit
  generic: Generic.Aux[A, L],
  size: hlist.Length.Aux[L, N],
  sizeToInt: nat.ToInt[N]
): SizeOf[A] =
  new SizeOf[A] {
    val value = sizeToInt.apply()
  }
```

次のようにしてコードをテストできる:

```tut:book:silent
case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

```tut:book
sizeOf[IceCream]
```
