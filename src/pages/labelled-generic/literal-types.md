## リテラル型

Scala の値は複数の型を持ちうる。
例えば、`"hello"` という文字列は少なくとも3つの型を持つ: `String`、`AnyRef`、そして`Any`だ[^multiple-inheritance]:

```tut:book
"hello" : String
"hello" : AnyRef
"hello" : Any
```

[^multiple-inheritance]:
`String` は `Serializable` や `Comparable` のような一束の他の型も持つが、ここでは無視しよう。

興味深いことに、`"hello"` はもう1つの型を持つ:
たった1つの値だけが属する、「シングルトン型(singleton type)」だ。
これは、コンパニオンオブジェクトを定義した際に得られるシングルトン型に似たものだ:

```tut:book:silent
object Foo
```

```tut:book
Foo
```

`Foo.type` という型は `Foo` の型であり、さらに `Foo` はその型の唯一の値である。

リテラル値が持つシングルトン型を **リテラル型(literal type)** と呼ぶ。
これは Scala に長い間存在していたが、コンパイラは、リテラルを最も近いシングルトン型でない型に「広げる」という振る舞いを持つため、我々は通常、それを操作することはない。
例えば、これら2つの式は本質的に等価である:

```tut:book
"hello"

("hello" : String)
```

Shapeless はリテラル型を扱うためのいくつかのツールを提供している。
まず、リテラルからなる式をシングルトン型のついたリテラル式に変換する、`narrow` というマクロがある:

```tut:book:silent
import shapeless.syntax.singleton._
```

```scala
var x = 42.narrow
// x: Int(42) = 42
```

ここでの `x` の型に注意しよう: `Int(42)` がリテラル型だ。
これは `42` という値だけを含む `Int` のサブ型である。
`x` に別の数値を代入しようとすると、コンパイルエラーとなる:

```scala
x = 43
// <console>:16: error: type mismatch:
//  found   : Int(43)
//  required: Int(42)
//        x = 43
//            ^
```

しかし、`x` は 通常のサブタイピング規則に従うので、やはり `Int` でもある。
`x` に対し操作を行うと、通常の型を持つ結果が得られる:

```tut:book:invisible
var x: 42 = 42
```

```tut:book
x + 1
```

Scala の任意のリテラルに対して `narrow` を用いることができる:

```scala
1.narrow
// res7: Int(1) = 1

true.narrow
// res8: Boolean(true) = true

"hello".narrow
// res9: String("hello") = hello

// などなど...
```

```tut:book:invisible
1 : 1
true : true
"hello" : "hello"
```

しかし、これを複合式で用いることはできない:

```scala
math.sqrt(4).narrow
// <console>:17: error: Expression scala.math.`package`.sqrt(4.0) does not evaluate to a constant or a stable reference value
//        math.sqrt(4.0).narrow
//                 ^
// <console>:17: error: value narrow is not a member of Double
//        math.sqrt(4.0).narrow
//                       ^
```

<div class="callout callout-info">
**Scala におけるリテラル型**

最近まで、Scala にはリテラル型を書くための構文が存在しなかった。
この型はコンパイラの中には存在していたが、それをコードの中で直接表現することはできなかった。
しかし、Lightbend Scala 2.12.1, Lightbend Scala 2.11.9, そして Typelevel Scala 2.11.8 以降には、リテラル型を直接サポートする構文がある。
これらのバージョンの Scala では、`-Yliteral-types` コンパイラオプションを使い、次のような宣言を書くことができる:

```tut:book
val theAnswer: 42 = 42
```

`42` という型は、上記の出力で見た `Int(42)` という型と同じものだ。
古い技術上の理由により、`Int(42)`という出力を見ることになるかもしれないが、今後の標準的な構文は `42` という形になる。
</div>

## 型タグ付けと幽霊型 {#sec:labelled-generic:type-tagging}

Shapeless はケースクラスのフィールド名をモデル化するためにリテラル型を用いている。
Shapeless は、フィールドの型にその名前を持つリテラル型の「タグ」を付けることで、これを達成している。
Shapeless がこれをどのように行っているのかを見ていく前に、どこにもタネや仕掛けがない(とはいえ、最小限の仕掛けはある…)ことを示すために、これを自分の手でやってみることにしよう。
ある数値があるとする:

```tut:book:silent
val number = 42
```

この数の`Int` という型は、2つの側面を持つ:
実行時には、実際の値と呼び出し可能なメソッドを持ち、
コンパイル時には、どれが一緒に動作することのできるコードなのかを計算したり、暗黙値の探索を行ったりするために、コンパイラが利用する。

「幽霊型(phantom type)」によってこの型に「タグ付け(tagging)」を行うことで、`number` の実行時の振る舞いを変えることなく、その型を変更できる。
幽霊型とは、以下のような、実行時には意味を持たない型のことだ:

```tut:book:silent
trait Cherries
```

`asInstanceOf` を用いて、`number` にタグを付けることができる。
結果として、この値はコンパイル時には `Int` と `Cherries` の両方の方を持ち、実行時には `Int` 型だけを持つことになる:

```tut:book
val numCherries = number.asInstanceOf[Int with Cherries]
```

Shapeless は、ADT のフィールドやサブ型に対し、その名前の情報を持つシングルトン型でタグを付けるためにこのトリックを用いている。
`asInstanceOf` を利用するのが気に食わないと思われたかもしれないが、心配はいらない:
shapeless はそのような気まずさを回避するため、2つのタグ付けのための構文を提供している。

1つ目の構文である `->>` は、矢印の右側の式に、その左側のリテラル式のシングルトン型によってタグ付けする:

```tut:book:silent
import shapeless.labelled.{KeyTag, FieldType}
import shapeless.syntax.singleton._

val someNumber = 123
```

```tut:book
val numCherries = "numCherries" ->> someNumber
```

ここでは、 `someNumber` に次のような幽霊型をタグ付けしている:

```scala
KeyTag["numCherries", Int]
```

このタグは対象のフィールドの名前とその型の両方を表現しており、この組み合わせは暗黙値の解決によって `Repr` 内の項目を探索する際に有用である。

2つ目の構文は、タグをリテラル値ではなく型として受け取る。
これは、どのタグを用いればいいかは分かっているものの、コード中に特定のリテラルを書くことができないときに役立つ:

```tut:book:silent
import shapeless.labelled.field
```

```tut:book
field[Cherries](123)
```

`FieldType` は、タグ付けされた型からタグとベース型を抽出する作業を単純化する型エイリアスである:

```scala
type FieldType[K, V] = V with KeyTag[K, V]
```

ひと目見て分かるように、shapeless はこのメカニズムをフィールドやサブ型に対してソースコード上の名前をタグ付けするのに利用している。

タグはコンパイル時にのみ存在し、実行時の表現は持たない。
これらを実行時に利用する値に変換するにはどうすればよいのだろうか?
Shapeless は、この目的のために `Witness` と呼ばれる型クラスを提供している[^witness]。
`Witness` と `FieldType` を組み合わせれば、非常に強力な能力を手に入れることができる---タグ付けされたフィールドからフィールド名を抽出する能力だ:

[^witness]: "witness" という用語は [数学の証明][link-witness] から借用したものだ。

```tut:book:silent
import shapeless.Witness
```

```tut:book
val numCherries = "numCherries" ->> 123
```

```tut:book:silent
// タグ付けされた値からタグを取得する
def getFieldName[K, V](value: FieldType[K, V])
    (implicit witness: Witness.Aux[K]): K =
  witness.value
```

```tut:book
getFieldName(numCherries)
```

```tut:book:silent
// タグ付けされた値からタグを外した値を取得する
def getFieldValue[K, V](value: FieldType[K, V]): V =
  value
```

```tut:book
getFieldValue(numCherries)
```

タグ付けされた要素からなる `HList` を構成すれば、`Map` のような性質を持つデータ構造を得ることができる。
タグによって、フィールドを参照したり、操作・置換したり、その型や名前の情報の履歴を管理したりすることができるようになる。
Shapeless では、このような構造を「レコード(record)」と呼ぶ。

### レコードと *LabelledGeneric*

レコードは、タグ付けされた要素からなる `HList` である:

```tut:book:silent
import shapeless.{HList, ::, HNil}
```

```tut:book
val garfield = ("cat" ->> "Garfield") :: ("orange" ->> true) :: HNil
```

`garfield` の型を明示すると、次のようになる:

```scala
// FieldType["cat",    String]  ::
// FieldType["orange", Boolean] ::
// HNil
```

ここでは、レコードについてこれ以上深く立ち入ることはしない。レコードが `LabelledGeneric` のジェネリック表現であるということを知ってもらえれば十分だ。
`LabelledGeneric` は、積型や余積型の各要素に、具体的な ADT 上の対応するフィールドや型の名前のタグを付けたものだ(ただし、名前は `String` ではなく `Symbol` として表現される)。
Shapeless は、[@sec:ops:record]節で説明するような、`Map` のものに似た一連の操作を提供している。
けれども、今は、`LabelledGeneric` を用いた型クラスの導出について見ていくことにしよう。
