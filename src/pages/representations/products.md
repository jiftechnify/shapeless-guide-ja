## 積型のジェネリックな表現

前節では、積の型のジェネリック表現としてタプルを導入した。
残念ながら、Scala の組み込みのタプル型には、shapeless の目的に適さない、いくつかの欠点がある:

 1. タプルは要素数ごとにそれぞれ異なる、関連のない型を持つので、要素数を抽象化したコードを書くのが難しくなっている。

 2. フィールドを持たない積を表現するのに重要な役割を果たす、長さ0のタプルの型が存在しない。
    議論の余地はあるが、`Unit`を使うこともできるだろう。しかし、理想的にはすべてのジェネリック表現は意味のある共通のスーパー型を持つべきである。
    `Unit` と `Tuple2` の最小上界(共通の親クラスのなかで最も下位のもの)は `Any` なので、これらの2つを組み合わせるのは実用的でない。

これらの理由のため、shapeless は **不均質リスト(heterogeneous list)** または `HList` と呼ばれる、積型に対する別のジェネリック表現を利用している。

[^hlist-name]: `Product` は `HList` よりも良い名前かもしれないが、残念なことに標準ライブラリには既に `scala.Product` 型が存在している。

`HList` は 空のリスト `HNil` か、任意の型 `H` と別の `HList` である `T` の組 `::[H, T]` のいずれかである。
それぞれの `::` がそれぞれの `H` と `T` を持つので、各要素の型はリスト全体の型とは別に表現される:

```tut:book:silent
import shapeless.{HList, ::, HNil}

val product: String :: Int :: Boolean :: HNil =
  "Sunday" :: 1 :: false :: HNil
```

上の `HList` において、その型と値は鏡合わせとなっている。
いずれも `String`、`Int`、そして `Boolean` の3つのメンバーを持つ。
`head` や `tail` を取得でき、このとき要素の型は保存される:

```tut:book
val first = product.head
val second = product.tail.head
val rest = product.tail.tail
```

コンパイラはそれぞれの `HList` の正確な長さを知っているので、空リストの `head` や `tail` を取得しようとするとコンパイルエラーとなる:

```tut:book:fail
product.tail.tail.tail.head
```

要素の検査や探索に加え、`HList` を操作したり変換したりすることもできる。
例えば、`::` メソッドを用いて要素を先頭に追加することができる。
やはり、結果の型が要素の数と型を反映していることに注目してほしい:

```tut:book:silent
val newProduct = 42L :: product
```

Shapeless は、リストの変換、フィルタ、連結のようなより複雑な演算を行うためのツールも提供している。
これらの詳細については、第2部で説明する。

ここまで見てきた `HList` の振る舞いはマジックなどではない。
これらすべての機能は、 `::` と `HNil` のかわりに `(A, B)` と `Unit` を用いても達成できるだろう。
しかし、アプリケーションで使われているセマンティックな型とそのジェネリック表現の型を分離しておくことには利点がある。
`HList` はこの「分離」をもたらすのだ。

### *Generic* を用いた表現の切り替え

Shapeless は `Generic` と呼ばれる、具体的な ADT とそのジェネリック表現を相互に切り替えることを可能にする型クラスを提供している。
舞台の裏側におけるマクロのマジックのおかげで、ボイラープレートなしで `Generic` のインスタンスを召喚できるようになっている:

```tut:book:silent
import shapeless.Generic

case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

```tut:book
val iceCreamGen = Generic[IceCream]
```

`Generic` のインスタンスは、そのジェネリック表現の型を保持する `Repr` 型メンバーを持つことに注意してほしい。
この場合、`iceCreamGen.Repr` は `String :: Int :: Boolean :: HNil` である。
`Generic` のインスタンスは2つのメソッドを持つ:
ひとつは `Repr` 型 **への** 変換を行う `to` 、もうひとつは `Repr` 型 **からの** 変換を行う `from` だ:

```tut:book
val iceCream = IceCream("Sundae", 1, false)

val repr = iceCreamGen.to(iceCream)

val iceCream2 = iceCreamGen.from(repr)
```

2つの ADT が同じ `Repr` 型を持つ場合、`Generic` を用いてそれらを相互に変換することができる:

```tut:book:silent
case class Employee(name: String, number: Int, manager: Boolean)
```

```tut:book
// アイスクリームから従業員を生成する
val employee = Generic[Employee].from(Generic[IceCream].to(iceCream))
```

<div class="callout callout-info">
**その他の積型**

注目すべきことに、Scala のタプルは実のところケースクラスなので、`Generic` はタプルも正しく扱うことができる:

```tut:book:silent
val tupleGen = Generic[(String, Int, Boolean)]
```

```tut:book
tupleGen.to(("Hello", 123, true))
tupleGen.from("Hello" :: 123 :: true :: HNil)
```

22個より多くのフィールドを持つケースクラスに対しても動作する:

```tut:book:silent
case class BigData(
  a:Int,b:Int,c:Int,d:Int,e:Int,f:Int,g:Int,h:Int,i:Int,j:Int,
  k:Int,l:Int,m:Int,n:Int,o:Int,p:Int,q:Int,r:Int,s:Int,t:Int,
  u:Int,v:Int,w:Int)
```

```tut:book
Generic[BigData].from(Generic[BigData].to(BigData(
  1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23)))
```

バージョン2.10以前の Scala には、ケースクラスのフィールド数に22個までという上限が存在した。この制限は2.11で名目上撤廃されたが、`HList` を用いれば、いまだに残っている [Scala における22フィールド制限][link-dallaway-twenty-two] を回避できる。

</div>
