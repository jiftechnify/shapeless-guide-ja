## 多相的関数

Shapeless は、結果の型が引数の型に依存するような **多相的関数** を表現する、 `Poly` という型を提供している。
ここではその動作について簡単な説明を行う。
次の節に含まれるのは実際の shapeless コードではないことに注意してほしい---説明用に単純化した API を作るために、実際の shapeless の `Poly` が持つ柔軟性や使いやすさにについては無視する。

### *Poly* はいかにして動くのか

根本的には、`Poly` はジェネリックな `apply` メソッドを持つオブジェクトである。
`A`型の通常の引数に加え、`Poly` は `Case[A]` 型の暗黙の引数を受け取る:

```tut:book:silent
// これは実際の shapeless のコードではない。
// 説明のためのコードである。

trait Case[P, A] {
  type Result
  def apply(a: A): Result
}

trait Poly {
  def apply[A](arg: A)(implicit cse: Case[this.type, A]): cse.Result =
    cse.apply(arg)
}
```

実際の `Poly` を定義するには、関心のある引数の型のそれぞれに対して `Case` のインスタンスを提供する。
それらが、実際の関数の本体の実装となる:

```tut:book:silent
// これは実際の shapeless のコードではない。
// 説明のためのコードである。

object myPoly extends Poly {
  implicit def intCase =
    new Case[this.type, Int] {
      type Result = Double
      def apply(num: Int): Double = num / 2.0
    }

  implicit def stringCase =
    new Case[this.type, String] {
      type Result = Int
      def apply(str: String): Int = str.length
    }
}
```

`myPoly.apply` を呼び出すと、いつものようにコンパイラが対応する `Case` を探し出し、暗黙の引数として挿入する:

```tut:book
myPoly.apply(123)
```

コンパイラが `Case` のインスタンスを追加のインポートなしで見つけられるように、ここでは繊細なスコーピングの振る舞いを利用している。
`Case` は `Poly` のシングルトン型を参照する余分な型パラメータ `P` を持つ。
`Case[P, A]` の暗黙のスコープには、`Case`、`P`、そして `A` のコンパニオンオブジェクトが含まれる。
ここでは `P` に `myPoly.type` を代入している。`myPoly.type` のコンパニオンオブジェクトとは、`myPoly` それ自身である。
言い換えれば、`Poly` の本体の中で定義された `Case` は、呼び出し場所にかかわらず常にスコープの中に含まれる。

### *Poly* の構文

上記のコードは実際の shapeless のコードではない。
幸い、shapeless では `Poly` をもっと簡単に定義できるようになっている。
正しい構文によって書き直した `myPoly` 関数は以下の通りだ:

```tut:book:silent
import shapeless._

object myPoly extends Poly1 {
  implicit val intCase: Case.Aux[Int, Double] =
    at(num => num / 2.0)

  implicit val stringCase: Case.Aux[String, Int] =
    at(str => str.length)
}
```

いくつか、先程の偽の構文との重要な違いがある:

 1. `Poly` ではなく、`Poly1` という名前のトレイトを継承している。
    Shapeless は `Poly` 型と、`Poly1` から `Poly22` までの一連のサブ型を持つ。これは様々な数の引数を持つ多相的関数をサポートするためのものである。

 2. `Case.Aux` 型は `Poly` のシングルトン型を参照していないように見える。
    `Case.Aux` は、実際には `Poly1` の本体の中で定義された型エイリアスである。
    シングルトン型はそこにある---ただ、我々には見えないだけだ。

 3. 各場合を定義する際に、`at` というヘルパーメソッドを用いている。
    これは[@sec:generic:idiomatic-style]節で説明したインスタンスのコンストラクタメソッドとして働き、多くのボイラープレートを削減する。

構文的な違いを除けば、shapeless で書いた `myPoly` は偽のバージョンと機能的に同一である。
`Int` または `String` の引数とともにこれを呼び出すと、対応する返り値型の結果が得られる:

```tut:book
myPoly.apply(123)
myPoly.apply("hello")
```

Shapeless は 2つ以上の引数を持つ `Poly` もサポートしている。
2引数の例を挙げる:

```tut:book:silent
object multiply extends Poly2 {
  implicit val intIntCase: Case.Aux[Int, Int, Int] =
    at((a, b) => a * b)

  implicit val intStrCase: Case.Aux[Int, String, String] =
    at((a, b) => b * a)
}
```

```tut:book
multiply(3, 4)
multiply(3, "4")
```

`Case` は単なる暗黙の値なので、型クラスに基づくケースを定義し、これまでの章で説明したすべての暗黙値の解決を応用できる。
異なる文脈の中にある数値を合計する、単純な例を以下に示す:

```tut:book:silent
import scala.math.Numeric

object total extends Poly1 {
  implicit def base[A](implicit num: Numeric[A]):
      Case.Aux[A, Double] =
    at(num.toDouble)

  implicit def option[A](implicit num: Numeric[A]):
      Case.Aux[Option[A], Double] =
    at(opt => opt.map(num.toDouble).getOrElse(0.0))

  implicit def list[A](implicit num: Numeric[A]):
      Case.Aux[List[A], Double] =
    at(list => num.toDouble(list.sum))
}
```

```tut:book
total(10)
total(Option(20.0))
total(List(1L, 2L, 3L))
```

<div class="callout callout-warning">
**型推論の怪**

`Poly` は Scala の型推論を安全地帯から追いやる。
コンパイラにたくさんの推論を一度に行わせれば、コンパイラを容易く混乱させることができてしまう。
例えば、次のコードはコンパイルを通る:

```tut:book:silent
val a = myPoly.apply(123)
val b: Double = a
```

しかし、2つの行をまとめると、コンパイルエラーを引き起こす:

```tut:book:fail
val a: Double = myPoly.apply(123)
```

型アノテーションを追加すれば、コードは再びコンパイルを通るようになる:

```tut:book
val a: Double = myPoly.apply[Int](123)
```

この振る舞いは紛らわしく、鬱陶しいものだ。
残念ながら、この問題を回避するための具体的な規則のようなものは存在しない。
唯一の一般的なガイドラインは、コンパイラに過剰な制約を書けないように心がけ、一度に1つの制約を解決させるようにし、コンパイラが行き詰まったら手がかりを与える、というものだ。
</div>
