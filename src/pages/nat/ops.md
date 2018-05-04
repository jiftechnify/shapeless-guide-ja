## *Nat* に関するその他の演算

Shapeless は他にも `Nat` に基づく一連の演算を提供している。
`HList` や `Coproduct` の `apply` メソッドは、`Nat` を値パラメータまたは型パラメータとして受け取ることができる:

```tut:book:silent
import shapeless._

val hlist = 123 :: "foo" :: true :: 'x' :: HNil
```

```tut:book
hlist.apply[Nat._1]
hlist.apply(Nat._3)
```

`take`、`drop`、`slice`、そして `updatedAt` のような演算も用意されている:

```tut:book
hlist.take(Nat._3).drop(Nat._1)
hlist.updatedAt(Nat._1, "bar").updatedAt(Nat._2, "baz")
```

これらの演算や、それに対応する型クラスは、積型や余積型の中の個々の要素を操作するのに有用だ。
