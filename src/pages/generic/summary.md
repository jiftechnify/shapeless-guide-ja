## Summary

本章では、型クラスのインスタンスを自動的に導出するために、`Generic`、`HList`、そして `Coproduct` をどう利用すればいいかについて説明した。
また、複雑・再帰的な型を処理するための手段である `Lazy` 型についても説明した。
これらのすべてを考慮に入れた、型クラスのインスタンスの自動導出のための共通の「スケルトン」は以下のようなものとなる。

まず、型クラスを定義する:

```tut:book:silent
trait MyTC[A]
```

基本となるインスタンスを定義する:

```tut:book:silent
implicit def intInstance: MyTC[Int] = ???
implicit def stringInstance: MyTC[String] = ???
implicit def booleanInstance: MyTC[Boolean] = ???
```

`HList` に対するインスタンスを定義する:

```tut:book:silent
import shapeless._

implicit def hnilInstance: MyTC[HNil] = ???

implicit def hlistInstance[H, T <: HList](
  implicit
  hInstance: Lazy[MyTC[H]], // Lazy で包む
  tInstance: MyTC[T]
): MyTC[H :: T] = ???
```

必要ならば、`Coproduct` に対するインスタンスを定義する:

```tut:book:silent
implicit def cnilInstance: MyTC[CNil] = ???

implicit def coproductInstance[H, T <: Coproduct](
  implicit
  hInstance: Lazy[MyTC[H]], // Lazy で包む
  tInstance: MyTC[T]
): MyTC[H :+: T] = ???
```

最後に、`Generic` に対するインスタンスを定義する:

```tut:book:silent
implicit def genericInstance[A, R](
  implicit
  generic: Generic.Aux[A, R],
  rInstance: Lazy[MyTC[R]] // Lazy 包む
): MyTC[A] = ???
```

次章では、このようなスタイルでコードを書く際の助けとなる、いくつかの有用な理論やプログラミングパターンについて説明する。
[@sec:labelled-generic]章では、型クラスの導出について再考する。ここでは、ADT のフィールドや型の名前を調べることが可能な `Generic` の変種を用いる。
