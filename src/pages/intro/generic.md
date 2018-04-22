## ジェネリックプログラミングとは何か?

型が役に立つのは、それが具体的だからだ: 型は別々のコード片がうまく噛み合うかどうかを示し、バグを防ぎ、コードを書く際に解への道標となる。

しかし、時に型は具体的 **すぎる**。
繰り返しを避けるために、型の間の類似性を利用したい状況もある。
例えば、次のような定義を考えてみよう:

```tut:book:silent
case class Employee(name: String, number: Int, manager: Boolean)

case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

これらの2つのケースクラスは違った種類のデータを表現している。しかし、これらの間には明らかな類似点がある: どちらも同じ型の3つのフィールドを持っているのだ。
CSV ファイルにシリアライズする、といったジェネリックな操作を実装したいとしよう。
2つの型が似ているにもかかわらず、2つの別々のシリアライズメソッドを書かなければならない:

```tut:book:silent
def employeeCsv(e: Employee): List[String] =
  List(e.name, e.number.toString, e.manager.toString)

def iceCreamCsv(c: IceCream): List[String] =
  List(c.name, c.numCherries.toString, c.inCone.toString)
```

ジェネリックプログラミングは、このような違いを克服するためのものである。
Shapeless を用いれば、特定の型を、共通のコードで操作できるようなジェネリックな型へと便利に変換できる。

例えば、以下のようなコードを利用して、従業員(Employee)とアイスクリーム(IceCream)を同じ型の値に変換することができる。
この例についていけなくても心配はいらない---様々な概念に対しては、この後で取り組むことにしよう:

```tut:book:silent
import shapeless._
```

```tut:book
val genericEmployee = Generic[Employee].to(Employee("Dave", 123, false))
val genericIceCream = Generic[IceCream].to(IceCream("Sundae", 1, false))
```

どちらの値も同じ型を持っている。
これらはどちらも、`String`、`Int`、そして`Boolean`を1つずつ持つ「不均質リスト」(heterogeneous list、 略して`HList`)である。
`HList`とそれが果たす重要な役割についてはこのあとすぐに見ていく。
ここでのポイントは、これらの値を同じ関数でシリアライズできるということだ:

```tut:book:silent
def genericCsv(gen: String :: Int :: Boolean :: HNil): List[String] =
  List(gen(0), gen(1).toString, gen(2).toString)
```

```tut:book
genericCsv(genericEmployee)
genericCsv(genericIceCream)
```

これは基本的な例だが、ジェネリックプログラミングの本質へ向かうための手がかりとなる。
ジェネリックな構成要素を利用して解決できるように問題を再定義し、幅広い型を扱う小さな「核」のようなコードを書く。
shapeless を利用したジェネリックプログラミングによって、大量のボイラープレートを削減することができ、その結果 Scala アプリケーションは読みやすく、書きやすく、そして保守しやすいものとなる。

すごいと思っただろう? 思ったとおりだ。 さあ、飛び込もう!
