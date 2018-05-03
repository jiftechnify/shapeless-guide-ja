## レコードの操作 {#sec:ops:record}

本章では、`shapeless.ops.hlist` と `shapeless.ops.coproduct` パッケージの型クラスを見ていくのに時間をとってきた。
3つ目の重要なパッケージ: `shapeless.ops.record` について言及せずに終わるわけにはいかない。

Shapeless の「レコード演算」は、タグ付けされた要素からなる `HList` の上の、 `Map` に似た操作を提供している。
アイスクリームを含む、ひと握りの例を挙げる:

```tut:book:silent
import shapeless._

case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

```tut:book
val sundae = LabelledGeneric[IceCream].
  to(IceCream("Sundae", 1, false))
```

`HList` や `Coproduct` の既に見た演算とは違って、レコード演算の構文を利用するには `shapeless.record` から明示的にインポートする必要がある:

```tut:book:silent
import shapeless.record._
```

### フィールドの選択

`get` 拡張メソッドと、それに対応する `Selector` 型クラスは、タグによるフィールドの取り出しを可能にする:

```tut:book
sundae.get('name)
```

```tut:book
sundae.get('numCherries)
```

定義されていないフィールドにアクセスしようとすると、期待通りにコンパイルエラーとなる:

```tut:book:fail
sundae.get('nomCherries)
```

### フィールドの更新と削除

`updated` メソッドと `Updater` 型クラスは、キーによるフィールドの変更を可能にする。
`remove` メソッドと `Remover` 型クラスは、キーによるフィールドの削除を可能にする:

```tut:book
sundae.updated('numCherries, 3)
```

```tut:book
sundae.remove('inCone)
```

`updateWith` メソッドと `Modifier` 型クラスは、更新関数によるフィールドの変更を可能にする:

```tut:book
sundae.updateWith('name)("MASSIVE " + _)
```

### 通常の *Map* への変換

`toMap` メソッドと `ToMap` 型クラスは、レコードの `Map` への変換を可能にする:

```tut:book
sundae.toMap
```

### その他の操作
他にも、余裕がないために説明できなかったレコードの操作がある。
フィールド名の変更、レコードのマージ、値の変換なども可能だ。
詳しくは、`shapeless.ops.record` と `shapeless.syntax.record` のソースコードを参照してほしい。
