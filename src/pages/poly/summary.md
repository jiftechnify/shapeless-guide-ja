## まとめ

本章では、引数の型によって返り値の型が変わるような **多相的関数** について説明した。
Shapeless の `Poly` がどのように定義されているのかを見た後、それが `map`、`flatMap`、`foldLeft`、そして `foldRight`のような関数的演算の実装においてどのように利用されているのかを見た。

それぞれの操作は `HList` 上の拡張メソッドとして実装されており、これは `Mapper`、`FlatMapper`、`LeftFolder` などの、対応する型クラスに基づいている。
これらの型クラス、`Poly`、そして[@sec:type-level-programming:chaining]節のテクニックを用いて、高度な変換の列を含む独自の型クラスを作成できる。
