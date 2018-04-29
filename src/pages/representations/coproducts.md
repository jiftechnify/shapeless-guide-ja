## ジェネリックな余積型

Shapeless が積型をエンコードする方法については分かった。
余積についてはどうだろうか?
先程 `Either` を見てきたが、これはタプルと同様の欠点を抱えている。
やはり、shapeless は `HList` に似た独自の表現を提供している:

```tut:book:silent
import shapeless.{Coproduct, :+:, CNil, Inl, Inr}

case class Red()
case class Amber()
case class Green()

type Light = Red :+: Amber :+: Green :+: CNil
```

一般的に余積は `A :+: B :+: C :+: CNil` という形をとり、これは「A または B または C 」を意味する。ここで `:+:` は大雑把にいえば `Either` として解釈できる。
余積全体の型は、可能性のあるすべての型を含む「選言(disjunction)」として表現されるが、この型の具体的なインスタンスはただ1つの可能な型の値を持つ。
`:+:` は `Inl` と `Inr` の2つのサブ型を持ち、これらはそれぞれ大雑把に `Left` と `Right` に対応する。
`Inl` と `Inr` のコンストラクタをネストすることで、余積のインスタンスを生成できる:

```tut:book
val red: Light = Inl(Red())
val green: Light = Inr(Inr(Inl(Green())))
```

すべての余積型は `CNil` を終端に持つ。これは `Nothing` と同様の、値をもたない空の型である。
`CNil` のインスタンスを生成したり、`Inr` のインスタンスのみから `Coproduct` を構築することはできない。
余積型の値には、必ずちょうど1つの `Inl` が含まれる。

`Coproduct` もまた、特段変わったものではないということを述べておきたい。
上記のような機能は `:+:` と `CNil` のかわりに `Either` と `Nothing`を利用しても達成できる。
`Nothing` を用いるのには技術的な困難があるが、`CNil` のかわりに他の値を持たない型や任意のシングルトン型を利用することもできるだろう。

### *Generic* を用いた表現の切り替え

`Coproduct` の型を一目で分析するのは難しい。
しかし、それがより広範なジェネリック表現という概念にいかにして噛み合うかを見ることはできる。
ケースクラスやケースオブジェクトに加え、shapeless の `Generic` 型クラスは sealed トレイトや抽象クラスもまた理解することができる:

```tut:book:silent
import shapeless.Generic

sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape
```

```tut:book
val gen = Generic[Shape]
```

`Shape` に対する `Generic` インスタンスの `Repr` の型は、次のような、sealed トレイトのサブ型を含む `Coproduct` となる:
`Rectangle :+: Circle :+: CNil`
ジェネリック表現の `to` や `from` メソッドを利用し、`Shape` と `gen.Repr` の間の相互変換を行うことができる:

```tut:book
gen.to(Rectangle(3.0, 4.0))

gen.to(Circle(1.0))
```
