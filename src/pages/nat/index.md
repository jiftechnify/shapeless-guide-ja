# 型で数える {#sec:nat}

時に、型レベルでものを数えなければならないことがある。
例えば、`HList` の長さや、計算中これまでに展開した項の数を知る必要が出てくるかもしれない。
数値を値として表現するのは十分容易だが、暗黙値の解決に影響を与えたければ、数を型のレベルで表現する必要がある。
本章では型による計数の背景にある理論について説明し、それの型クラス導出へのいくつかの強力な応用を紹介する。


## 数値を型として表現する

Shapeless は、型レベルで自然数を表現するために「チャーチ表現(Church encoding)」を利用している。
Shapeless は、2つのサブ型を持つ `Nat` 型を提供している:
ゼロを表現する `_0` と、`N+1` を表現する `Succ[N]`だ:

```tut:book:silent
import shapeless.{Nat, Succ}

type Zero = Nat._0
type One  = Succ[Zero]
type Two  = Succ[One]
// 以下同様...
```

Shapeless は、`Nat._N` という形で、`Nat` の最初の22個に対するエイリアスを提供している:

```tut:book:silent
Nat._1
Nat._2
Nat._3
// 以下同様...
```

`Nat` は実行時のセマンティクスを持たない。
`Nat` を実行時の `Int` に変換するには、`ToInt` 型クラスを利用する必要がある:

```tut:book:silent
import shapeless.ops.nat.ToInt

val toInt = ToInt[Two]
```

```tut:book
toInt.apply()
```

`Nat.toInt` メソッドは、`toInt.apply()` の呼び出しの便利な省略形である。
これは `ToInt` のインスタンスを暗黙の引数として受け取る:

```tut:book
Nat.toInt[Nat._3]
```
