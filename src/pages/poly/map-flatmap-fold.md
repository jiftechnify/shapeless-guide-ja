## *Poly* を用いた map, flatMap

Shapeless は `Poly` に基づく一連の関数的演算を提供している。これらはそれぞれ演算型クラスとして実装されている。
例として、`map` と `flatMap` を見ていこう
`map` は以下のようになっている:

```tut:book:silent
import shapeless._

object sizeOf extends Poly1 {
  implicit val intCase: Case.Aux[Int, Int] =
    at(identity)

  implicit val stringCase: Case.Aux[String, Int] =
    at(_.length)

  implicit val booleanCase: Cuse.Aux[Boolean, Int] =
    at(bool => if(bool) 1 else 0)
}
```

```tut:book
(10 :: "hello" :: true :: HNil).map(sizeOf)
```

結果の `HList` に含まれる要素は、`sizeOf` の各 `Case` に対応した型を持っていることに注意してほしい。
最初の `HList` のすべての要素に対応する `Case` を含むような任意の `Poly` によって、 `map` を行うことができる。
コンパイラが、ある要素に対応する `Case` を見つけられなかった場合は、コンパイルエラーとなる:

```tut:book:fail
(1.5 :: HNil).map(sizeOf)
```

`Poly` の対応するすべてのケースが他の `HList` を返すならば、`HList` の上で `flatMap` を行うこともできる:

```tut:book:silent
object valueAndSizeOf extends Poly1 {
  implicit val intCase: Case.Aux[Int, Int :: Int :: HNil] =
    at(num => num :: num :: HNil)

  implicit val stringCase: Case.Aux[String, String :: Int :: HNil] =
    at(str => str :: str.length :: HNil)

  implicit val booleanCase: Case.Aux[Boolean, Boolean :: Int :: HNil] =
    at(bool => bool :: (if(bool) 1 else 0) :: HNil)
}
```

```tut:book
(10 :: "hello" :: true :: HNil).flatMap(valueAndSizeOf)
```

足りないケースがあったり、どれかのケースが `HList` を返さない場合、やはりコンパイルエラーとなる:

```tut:book:fail
// 適切でない Poly を用いた flatMap:
(10 :: "hello" :: true :: HNil).flatMap(sizeOf)
```

`map` と `flatMap` は、それぞれ `Mapper`、`FlatMapper` と呼ばれる型クラスに基づいている。
[@sec:poly:product-mapper]節では、`Mapper` を直接利用する例を見ていく。

## *Poly* を用いた畳み込み

`map` や `flatMap` に加え、shapeless は `Poly2` に基づく `foldLeft` と `foldRight` 演算も提供している:

```tut:book:silent
import shapeless._

object sum extends Poly2 {
  implicit val intIntCase: Case.Aux[Int, Int, Int] =
    at((a, b) => a + b)

  implicit val intStringCase: Case.Aux[Int, String, Int] =
    at((a, b) => a + b.length)
}
```

```tut:book
(10 :: "hello" :: 100 :: HNil).foldLeft(0)(sum)
```

`reduceLeft`、`reduceRight`、`foldMap` なども可能だ。
各演算が、それぞれの型クラスと対応している。
利用できるすべての演算の調査については、読者への課題とする。
