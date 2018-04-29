## 復習: 代数的データ型

**代数的データ型(algebraic data type, ADT)**[^adts]は、耳慣れない名前だが、非常にシンプルな意味を持つ関数プログラミングの概念である。
これは、データを「and」と「or」を用いて表現するための、(関数プログラミングにおいて)慣用的な(idiomatic な)手法である。例えば:

 - 図形(shape) は、 長方形(rectangle) **または(or)** 円形(circle) である
 - 長方形は、幅 **と(and)** 高さをもつ
 - 円形は、半径をもつ

[^adts]: 「抽象データ型(abstract data type)」と混同しないこと。これは、ここで説明している概念とはほとんど関連のない、まったく別のコンピュータサイエンスのツールである。

ADT の用語では、長方形や円形のような「and」を含む型を **積(product)** と呼び、図形のような「or」を含む型を **余積(coproduct)** と呼ぶ。
Scala においては、多くの場合、積はケースクラスで、余積は sealed トレイトで表現する:

```tut:book:silent
sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape

val rect: Shape = Rectangle(3.0, 4.0)
val circ: Shape = Circle(1.0)
```

ADT の美しさは、それが完全に型安全であるところにある。
コンパイラは、我々が定義した代数(algebra)[^algebra]に関する完全な知識を持つので、独自の型を含む、完全で正しく型付けられたメソッドを書く助けになってくれる:

[^algebra]: 「代数」という言葉は、長方形(rectangle)・円形(circle)のような我々が定義したシンボルと、メソッドの形で表現される、それらのシンボルに対する操作の規則のことを意味する。

```tut:book:silent
def area(shape: Shape): Double =
  shape match {
    case Rectangle(w, h) => w * h
    case Circle(r)       => math.Pi * r * r
  }
```

```tut:book
area(rect)
area(circ)
```

### もうひとつの表現

Sealed トレイトとケースクラスが Scala における ADT の表現として最も便利な形であることには疑いの余地はない。
しかし、それが **唯一の** 表現だというわけではない。
例えば、Scala の標準ライブラリは `Tuple` という形でジェネリックな積の型を、`Either` という形でジェネリックな余積の型を提供している。
`Shape` を表現するのに、これらを使うこともできただろう:

```tut:book:silent
type Rectangle2 = (Double, Double)
type Circle2    = Double
type Shape2     = Either[Rectangle2, Circle2]

val rect2: Shape2 = Left((3.0, 4.0))
val circ2: Shape2 = Right(1.0)
```

この表現は上記のケースクラスによる表現よりも読むのが難しいが、それと同様の望ましい性質を持つ。
やはり、`Shape2` に関する完全に型安全な操作を書くことができる:

```tut:book:silent
def area2(shape: Shape2): Double =
  shape match {
    case Left((w, h)) => w * h
    case Right(r)     => math.Pi * r * r
  }
```

```tut:book
area2(rect2)
area2(circ2)
```

重要なことは、`Shape2` が `Shape` よりも **ジェネリックな** 表現であるということだ[^generic]。
`Double` の2つ組の上で動作するすべてのコードは `Rectangle2` の上でも動作し、その逆も成り立つ。
Scala 開発者として、我々は `Rectangle2` や `Circle2` のようなジェネリックな型よりも `Rectangle` や `Circle` のようなセマンティックな型を好む傾向がある。これは、ひとえに後者のほうがより特化した性質を持つためである。
しかし、一般的であるほうが望ましい場合もある。
例えば、データをディスクにシリアライズする場合は、`Double` の組と `Rectangle2` の間の違いは重要ではない。
2つの数値を書き込めば、それで十分なのだ。

Shapeless を使えば、これら2つの世界の良いところ取りができる:
普段は親しみやすいセマンティックな型を使い、相互運用性が必要になったらジェネリック表現に切り替える、といったことが可能となる(これについては後ほど説明する)。
しかし、shapelessは、ジェネリックな積や余積を表現するのに、`Tuple` や `Either` ではなく独自のデータ型を用いている。
次節ではその型を紹介する。

[^generic]: ここでは「ジェネリック」という言葉を、「型パラメータをもつ型」といった型にはまった意味ではなく、もっとくだけた意味で使っている。
