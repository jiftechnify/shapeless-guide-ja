## 復習: 型クラス {#sec:generic:type-classes}

インスタンスの導出について掘り下げていく前に、型クラスの重要な側面について少し復習をしておこう。

型クラスは、Haskell から輸入されたプログラミングパターンである(「クラス」という言葉は、オブジェクト指向プログラミングにおけるクラスとは関係ない)。
Scala では、これをトレイトと暗黙の値で表現する。
**型クラス** は、様々な型に適用できるような何らかの汎用的な機能を表現する、型パラメータをもつトレイトである:

```tut:book:silent
// A 型の値を CSV ファイルのセルからなる行に変換する
trait CsvEncoder[A] {
  def encode(value: A): List[String]
}
```

そして、関心のあるそれぞれの型に対する型クラスの **インスタンス** を実装する。
このインスタンスを自動的にスコープに入れたければ、それを型クラスのコンパニオンオブジェクトに配置すればよい。
または、ユーザが手動でインポートできるよう、別のライブラリのオブジェクトに配置することもできる:

```tut:book:silent
// 独自のデータ型:
case class Employee(name: String, number: Int, manager: Boolean)

// 独自のデータ型に対する CsvEncoder のインスタンス:
implicit val employeeEncoder: CsvEncoder[Employee] =
  new CsvEncoder[Employee] {
    def encode(e: Employee): List[String] =
      List(
        e.name,
        e.number.toString,
        if(e.manager) "yes" else "no"
      )
  }
```

各インスタンスには `implicit` キーワードで印を付けておき、そして対応する型の暗黙のパラメータを受け取る1つ以上のエントリーポイントとなるメソッドを定義する:

```tut:book:silent
def writeCsv[A](values: List[A])(implicit enc: CsvEncoder[A]): String =
  values.map(value => enc.encode(value).mkString(",")).mkString("\n")
```

テストデータを用いて `writeCsv` をテストしてみる:

```tut:book:silent
val employees: List[Employee] = List(
    Employee("Bill", 1, true),
    Employee("Peter", 2, false),
    Employee("Milton", 3, false)
)
```

`writeCsv` を呼び出すと、コンパイラは型パラメータの値を計算し、その型に対応する暗黙の `CsvEncoder` を探し出す:

```tut:book
writeCsv(employees)
```

スコープ内に対応する暗黙の `CsvEncoder` がある限り、どんなデータ型に対しても `writeCsv` を利用できる:

```tut:book:silent
case class IceCream(name: String, numCherries: Int, inCone: Boolean)

implicit val iceCreamEncoder: CsvEncoder[IceCream] =
  new CsvEncoder[IceCream] {
    def encode(i: IceCream): List[String] =
      List(
        i.name,
        i.numCherries.toString,
        if(i.inCone) "yes" else "no"
      )
  }

val iceCreams: List[IceCream] = List(
  IceCream("Sundae", 1, false),
  IceCream("Cornetto", 0, true),
  IceCream("Banana Split", 0, false)
)
```

```tut:book
writeCsv(iceCreams)
```

### インスタンスの解決

型クラスは非常に柔軟だが、関心のあるすべての型に対するインスタンスを定義する必要がある。
幸い、Scala コンパイラは、いくつかのユーザが定義した規則をもとにインスタンスの解決を行うという、隠された能力を持っている。
例えば、`A`型 と `B`型の `CsvEncoder` があるときに、`(A, B)`型に対する `CsvEncoder` を生成する、という規則を書くことができる:

```tut:book:silent
implicit def pairEncoder[A, B](
  implicit
  aEncoder: CsvEncoder[A],
  bEncoder: CsvEncoder[B]
): CsvEncoder[(A, B)] =
  new CsvEncoder[(A, B)] {
    def encode(pair: (A, B)): List[String] = {
      val (a, b) = pair
      aEncoder.encode(a) ++ bEncoder.encoder(b)
    }
  }
```

`implicit def` の すべての引数がそれ自体 `implicit` とマークされている場合、コンパイラはそれを他のインスタンスから新しいインスタンスを生成するための解決規則として用いる。
例えば、`writeCsv` に  `List[(Employee, IceCream)]` を渡して呼び出した場合、コンパイラは `pairEncoder`、`employeeEncoder`、`iceCreamEncoder` を組み合わせ、必要な `CsvEncoder[(Employee, IceCream)]` を作り出すことができる:

```tut:book
writeCsv(employees zip iceCreams)
```

`implicit val` や `implicit def` の形で表現された一連の規則が与えられると、コンパイラは、必要なインスタンスを作り出すためにそれらの組み合わせを **探索** することができる。
「暗黙値の解決(implicit resolution)」として知られるこの振る舞いが、Scala における型クラスパターンを非常に強力なものにしているのだ。

### 型クラス定義の慣習 {#sec:generic:idiomatic-style}

広く受け入れられている、慣習的な型クラスの定義スタイルとしては、いくつかの標準メソッドを含むコンパニオンオブジェクトを定義するというものがある:

```tut:book:silent
object CsvEncoder {
  // 「召喚」メソッド
  def apply[A](implicit enc: CsvEncoder[A]): CsvEncoder[A] =
    enc

  // 「コンストラクタ」メソッド
  def instance[A](func: A => List[String]): CsvEncoder[A] =
    new CsvEncoder[A] {
      def encode(value: A): List[String] =
        func(value)
    }

  // グローバルに公開された型クラスインスタンス
}
```

「召喚メソッド(summoner)」あるいは「具現化メソッド(materializer)」として知られる `apply` メソッドは、対象の型に対する型クラスのインスタンスを召喚するものだ:

```tut:book
CsvEncoder[IceCream]
```

単純な場合では、召喚メソッドは `scala.Predef` に定義されている `implicitly` メソッドと同じ働きをする:

```tut:book
implicitly[CsvEncoder[IceCream]]
```

しかし、[@sec:type-level-programming:depfun]節で見るように、shapeless を利用していると `implicitly` が型を正しく推論できない状況に陥ることがある。
正しい動作をする召喚メソッドを定義することはいつでもできるので、作成したすべての型クラスに対して自分で召喚メソッドを書く価値はある。
shapeless にある、"`the`"という名前の特殊なメソッドを利用することもできる(詳しくは後ほど説明する):

```tut:book:silent
import.shapeless._
```

```tut:book
the[CsvEncoder[IceCream]]
```

`pure` と名付けられることもある、`instance` メソッドは、無名クラス構文のボイラープレートを削減し、新しい型クラスインスタンスを簡潔に定義するための構文を提供する:

```tut:book:silent
implicit val booleanEncoder: CsvEncoder[Boolean] =
  new CsvEncoder[Boolean] {
    def encode(b: Boolean): List[String] =
      if(b) List("yes") else List("no")
  }
```

これはより短く書ける:

```tut:book:invisible
import CsvEncoder.instance
```

```tut:book:silent
implicit val booleanEncoder: CsvEncoder[Boolean] =
  instance(b => if(b) List("yes") else List("no"))
```

残念なことに、コードの組版の制約のせいで、たくさんのメソッドやインスタンスをもつ、長大なシングルトンを書くことができない。
そのため本書では、いくつかの定義を、文章の外側のコンパニオンオブジェクトに含めるようにしている。
読み進める際はこのことを心に留めておいてほしい。また、完全に動作する例を確認したい場合は、[@sec:intro:source-code]節のリンク先にある補足リポジトリを参照してほしい。
