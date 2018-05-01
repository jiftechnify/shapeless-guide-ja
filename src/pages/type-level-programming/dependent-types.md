## 依存型(dependent type)

前章では、ADT 型をジェネリック表現に変換する型クラスである `Generic` の利用に多くの時間を費やしてきた。
しかし、`Generic` をはじめ shapeless の多くの部分を支えている重要な小さな理論: **依存型(dependent type)** についてはまだ説明していなかった。

これを説明するために、`Generic` をもっと詳しく見ていこう。
その簡素化された定義は以下のようなものだ:

```scala
trait Generic[A] {
  type Repr
  def to(value: A): Repr
  def from(value: Repr): A
}
```

`Generic` のインスタンスは2つの他の型を参照している:
型パラメータ `A` と 型メンバー `Repr` だ。
`getRepr` というメソッドを以下のように定義したとしよう。
返り値の型はどうなるだろうか?

```tut:book:silent
import shapeless.Generic

def getRepr[A](value: A)(implicit gen: Generic[A]) =
  gen.to(value)
```

その答えは、`gen` に渡されたインスタンスによって違ったものになる。
`getRepr` の呼び出しを展開する際、コンパイラは `Generic[A]` のインスタンスを探し出し、結果の型はそのインスタンスの中で定義された `Repr` の型となる:

```tut:book:silent
case class Vec(x: Int, y: Int)
case class Rect(origin: Vec, size: Vec)
```

```tut:book
getRepr(Vec(1, 2))
getRepr(Rect(Vec(0, 0), Vec(5, 5)))
```

ここで見たのが **依存型付け(dependent typing)** と呼ばれるものだ:
`getRepr` の結果の型は、それに渡された値パラメータの型メンバーに依存している。
`Repr` 型が、型メンバーのではなく、 `Generic` の型パラメータとして指定されていたと仮定しよう:

```tut:book:silent
trait Generic2[A, Repr]

def getRepr2[A, R](value: A)(implicit generic: Generic2[A, R]): R =
  ???
```

この場合、`getRepr` に所望の `Repr` の値を渡さなければならなかっただろう。これでは `getRepr` はほとんど役に立たないものになってしまう。
このことから得られる直感的な教訓は、型パラメータは「入力」として、型メンバーは「出力」として有用である、というものだ。
