## ソースコードと例 {#sec:intro:source-code}

本書はオープンソースである。
[Markdown ソース は Github 上にある][link-book-repo]。
本書はコミュニティによって定期的に更新されているので、最新バージョンを見るには、Github リポジトリを確認してほしい。

また、[Underscore の web サイト][link-underscore-book]では、印刷された本を手に入れることができる。
印刷版を入手した方には、更新版を公開する際にお知らせをお送りする。

[付随するリポジトリ][link-code-repo]には、主要な例の完全な実装が公開されている。
インストール方法の詳細については README を参照してほしい。
shapeless 2.3.2、および Typelevel Scala 2.11.8以降または Lightbend Scala 2.11.9/2.12.1以上の利用を前提としている。

本書に含まれる多くの例は、バージョン2.12.1の Typelevel Scala コンパイラでコンパイル・実行されている。
いくつかの改良点があるが、特にこのバージョンでは **型の中置表示(infix type printing)** が導入されている。これにより、REPL のコンソール出力が見やすくなっている:

```tut:book:invisible
import shapeless._
```

```tut:book
val repr = "Hello" :: 123 :: true :: HNil
```

古いバージョンの Scala を利用した場合、中置された型の表示は次のようになるだろう:

```scala
val repr = "Hello" :: 123 :: true :: HNil
// repr: shapeless.::[String,shapeless.::[Int,shapeless.::[Boolean,shapeless.HNil]]] = "Hello" :: 123 :: true :: HNil
```

落ち着こう!
結果の表示形式(中置か前置か)を除けば、これらの型は同じものだ。
前置の型を読むのが難しいと感じたら、Scala のバージョンをアップグレードすることをお勧めする。
単に `build.sbt` に以下のように書き加えればよい。バージョン番号は適切な最新のものに置き換えてほしい:

```scala
scalaOrganization := "org.typelevel"
scalaVersion      := "2.12.1"
```

`scalaOrganization` の設定は、 SBT 0.13.13 以降のみでサポートされている。
`project/build.properties` に次のように書くことで、SBT のバージョンを指定することができる(プロジェクトにこのファイルがなければ、新しく作成する):

```
sbt.version=0.13.13
```
