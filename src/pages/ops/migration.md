## 事例: ケースクラスのマイグレーション {#sec:ops:migration}

演算型クラスの力は、それらを我々自身のコードの構成要素として繋ぎ合わせたときに完全なものとなる。
強力な実例をみて、本章の結びとする:
ケースクラスの「マイグレーション」(または「進化」)を行うための型クラスだ[^database-migrations]。
例えば、我々のアプリケーションの第1バージョンが次のようなケースクラスを持っているとする:

[^database-migrations]: この用語は「データベースマイグレーション」(データベースのスキーマの更新を自動化する SQL スクリプト)から借りたものだ。

```tut:book:silent
case class IceCreamV1(name: String, numCherries: Int, inCone: Boolean)
```

マイグレーションライブラリは、特定の機械的な「更新」をタダで行えるようにする必要がある:

```tut:book:silent
// フィールドの削除:
case class IceCreamV2a(name: String, inCone: Boolean)

// フィールドの並べ替え
case class IceCreamV2b(name: String, inCone: Boolean, numCherries: Int)

// フィールドの挿入(デフォルト値を定めることができると過程する):
case class IceCreamV2c(
  name: String, inCone: Boolean, numCherries: Int, numWaffles: Int)
```

理想的には、次のようなコードが書けるようにしたい:

```scala
IceCreamV1("Sundae", 1, false).migrateTo[IceCreamV2a]
```

この型クラスは、追加のボイラープレートなしでマイグレーションの処理を行う必要がある。

### 型クラス

この`Migration` 型クラスは、変換元から変換先の型への変換を表現する。
これらの両方の型は導出の「入力」になるので、型パラメータとしてモデル化する。
公開するべき型メンバーはないので、`Aux` 型エイリアスは必要ない:

```tut:book:silent
trait Migration[A, B] {
  def apply(a: A): B
}
```

また、例を読みやすくするために、拡張メソッドを導入する:

```tut:book:silent
implicit class MigrationOps[A](a: A) {
  def migrateTo[B](implicit migration: Migration[A, B]): B =
    migration.apply(a)
}
```

### ステップ 1. フィールドの削除

さて、一つひとつ問題を解決していこう。まずフィールドの削除から始める。
いくつかのステップを踏むことで、これを解決できる:

 1. `A`型をジェネリック表現に変換する
 2. ステップ1の `HList` をフィルタする---`B`型にも含まれるフィールドのみを残す
 3. ステップ2の出力を `B`型に変換する

ステップ1、ステップ3は `Generic` または `LabelledGeneric` によって、またステップ2は `Intersection` と呼ばれる演算によって実装できる。
フィールドを名前で特定する必要があるので、`LabelledGeneric` を選んだほうが良さそうだ:

```tut:book:silent
import shapeless._
import shapeless.ops.hlist

implicit def genericMigration[A, B, ARepr <: HList, BRepr <: HList](
  implicit
  aGen  : LabelledGeneric.Aux[A, ARepr],
  bGen  : LabelledGeneric.Aux[B, BRepr],
  inter : hlist.Intersection.Aux[ARepr, BRepr, BRepr]
): Migration[A, B] = new Migration[A, B] {
  def apply(a: A): B =
    bGen.from(inter.apply(aGen.to(a)))
}
```

少し時間をとって、shapeless のコードベースから [`Intersection`][code-ops-hlist-intersection] を探し出してみよう。
この型クラスの `Aux` 型エイリアスは3つの型パラメータをとる:
2つの `HList` と共通部分(intersection)の出力の型だ。
上の例では、入力の型として `ARepr` と `BRepr` を、出力の型として `BRepr` を指定している。
これは、`B` が持つフィールドが `A` のフィールドの(名前と順序が等しいという意味で)厳密な部分集合であるときに限って暗黙値の解決が成功する、ということを意味する:

```tut:book
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2a]
```

`Migration` に適切でない型を与えて利用しようとすると、コンパイルエラーとなる:

```tut:book:fail
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2b]
```

### ステップ 2. フィールドの並べ替え

並べ替えをサポートするには、他の演算型クラスを頼る必要がある。
[`Align`][code-ops-hlist-align] 演算は、ある `HList` のフィールドを、もう1つの `HList` のフィールドの順番に合わせて並べ替えることを可能にする。
`Align` を用いて、我々のインスタンスを次のように再定義できる:

```tut:book:silent
implicit def genericMigration[
  A, B,
  ARepr <: HList, BRepr <: HList,
  Unaligned <: HList
](
  implicit
  aGen    : LabelledGeneric.Aux[A, ARepr],
  bGen    : LabelledGeneric.Aux[B, BRepr],
  inter   : hlist.Intersection.Aux[ARepr, BRepr, Unaligned],
  align   : hlist.Align[Unaligned, BRepr]
): Migration[A, B] = new Migration[A, B] {
  def apply(a: A): B =
    bGen.from(align.apply(inter.apply(aGen.to(a))))
}
```

並べ替える前の `ARepr` と `BRepr` の共通部分の型を表現するために `Unaligned` という名前の新しい型パラメータを導入し、`Unaligned` を `BRepr` に変換するために `Align` を用いる。
この修正された `Migration` の定義により、フィールドの削除と並べ替えの両方ができるようになった:

```tut:book
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2a]
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2b]

```

しかし、フィールドを追加しようとするとやはり失敗に終わる:

```tut:book:fail
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2c]
```

### ステップ 3. フィールドの追加

新しいフィールドの追加をサポートするには、デフォルト値を計算するメカニズムが必要となる。
Shapeless はこのための型クラスは提供していないが、Cats が `Monoid` という形でこれを提供している。
簡素化された定義はこうだ:

```scala
package cats

trait Monoid[A] {
  def empty: A
  def combine:(x: A, b: A): A
}
```

`Monoid` は2つの演算を定義している:
「ゼロ」値を生成する `empty` と、2つの値を「足し合わせる」ための `combine` だ。
我々のコードで必要なのは `empty` だけだが、`combine` の定義は自明だろう。

Cats は、関心のあるすべてのプリミティブ型(`Int`、`Double`、`Boolean`、`String`)に対する `Monoid` のインスタンスを提供している。
[@sec:labelled-generic]章のテクニックを利用して、`HNil` と `::` に対するインスタンスを定義することができる:

```tut:book:silent
import cats.Monoid
import cats.instances.all._
import shapeless.labelled.{field, FieldType}

def createMonoid[A](zero: A)(add: (A, A) => A): Monoid[A] =
  new Monoid[A] {
    def empty = zero
    def combine(x: A, y: A): A = add(x, y)
  }

implicit val hnilMonoid: Monoid[HNil] =
  createMonoid[HNil](HNil)((x, y) => HNil)

implicit def emptyHList[K <: Symbol, H, T <: HList](
  implicit
  hMonoid: Lazy[Monoid[H]],
  tMonoid: Monoid[T]
): Monoid[FieldType[K, H] :: T] =
  createMonoid(field[K](hMonoid.value.empty) :: tMonoid.empty) {
    (x, y) =>
      field[K](hMonoid.value.combine(x.head, y.head)) ::
        tMonoid.combine(x.tail, y.tail)
  }
```

`Migration` の最終的な実装を完成させるには、他のいくつかの演算と `Monoid` を組み合わせる(combine)[^monoid-pun]必要がある。
すべてのステップの一覧は次のとおりだ:

 1. `LabelledGeneric` を用いて `A`型をジェネリック表現に変換する
 2. `Intersection` を用いて `A` と `B` に共通のフィールドを持つ `HList` を計算する
 3. `B` にはあって、`A` にはないフィールドの型を求める
 4. `Monoid` を用いて、ステップ3で求めた型のデフォルト値を計算する
 5. ステップ2で求めた共通のフィールドのリストに、ステップ4で求めた新しいフィールドを追加する
 6. `Align` を用いて、ステップ5で求めたフィールドリストを `B` と同じ順番に並べ替える
 7. `LabelledGeneric` を用いて、ステップ6の結果を  `B`型に変換する

[^monoid-pun]: モノイドの「組み合わせ」演算(combine)とかかっていて面白い

ステップ1, 2, 4, 6, 7を実装する方法は既に見た。
ステップ3については `Diff` と呼ばれる `Intersection` とよく似た演算を、ステップ5については `Prepend` と呼ばれるもう1つの演算を用いて実装できる。
完全な実装は以下の通り:

```tut:book:silent
implicit def genericMigration[
  A, B, ARepr <: HList, BRepr <: HList,
  Common <: HList, Added <: HList, Unaligned <: HList
](
  implicit
  aGen    : LabelledGeneric.Aux[A, ARepr],
  bGen    : LabelledGeneric.Aux[B, BRepr],
  inter   : hlist.Intersection.Aux[ARepr, BRepr, Common],
  diff    : hlist.Diff.Aux[BRepr, Common, Added],
  monoid  : Monoid[Added],
  prepend : hlist.Prepend.Aux[Added, Common, Unaligned],
  align   : hlist.Align[Unaligned, BRepr]
): Migration[A, B] =
  new Migration[A, B] {
    def apply(a: A): B =
      bGen.from(align(prepend(monoid.empty, inter(aGen.to(a)))))
  }
```

このコードは、どの型クラスも値レベルでは利用していないことに注意してほしい。
`Added` データ型を計算するために `Diff` を用いているが、`diff.apply` を実行時に実際に呼び出す必要はない。
かわりに、`Added` のインスタンスを召喚するために `Monoid` を利用している。

この最終バージョンの型クラスがあれば、この事例研究の最初に設定したすべてのユースケースに対して `Migration` を利用できる:

```tut:book
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2a]
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2b]
IceCreamV1("Sundae", 1, true).migrateTo[IceCreamV2c]
```

演算型クラスによって作り出せるものは驚きに満ちている。
`Migration` は 1行の値レベルの実装を持つたった1つの `implicit def` しか持たない。
**1つの** 型の組を扱うために標準ライブラリを使って書くのと同じくらいの量のコードで、**すべての** ケースクラスの間のマイグレーションを自動化することが可能なのだ。
これが shapeless の力だ!
