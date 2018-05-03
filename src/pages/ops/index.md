# *HList* と *Coproduct* の操作 {#sec:ops}

第1部では、代数的データ型に対する型クラスインスタンスの導出を行う手法について説明した。
ほとんどすべての型クラスを強化するのに、型クラス導出を利用できる。しかし、より複雑なケースでは、`HList` や `Coproduct` を操作するためのたくさんのサポートコードを書く必要があるだろう。

第2部では、`shapeless.ops` パッケージについて見ていく。これは、プログラムの構成要素として利用できる一連の有用なツールを提供するものである。
各操作は2つの部分からなる:
暗黙値の解決に用いられる **型クラス** と、`HList` や `Coproduct` に対して呼び出すことができる **拡張メソッド(extension method)** だ。

3つのパッケージから利用できる、3つの一般的な操作のセットがある:

  - `shapeless.ops.hlist` は、`HList` に対する型クラスを定義している。
    これらは、`shapeless.syntax.hlist` で定義された `HList` 上の拡張メソッドを通して直接利用できる。

  - `shapeless.ops.product` は、`Coproduct` に対する型クラスを定義している。
    これらは、`shapeless.syntax.coproduct` で定義された `Coproduct` 上の拡張メソッドを通して直接利用できる。

  - `shapeless.ops.record` は、shapeless のレコード([@sec:labelled-generic:type-tagging]節で見た、タグ付けされた要素を含む `HList`)に対する型クラスを定義している。
    これらは、`shapeless.record` からインポートされ、`shapeless.syntax.record` で定義された `HList` 上の拡張メソッドを通して利用できる。

利用できるすべての操作を説明するだけの余裕はない。
幸い、多くの場合 shapeless のコードは理解しやすく、適切に文書化されている。
本書では、包括的な手引きを提供するのではなく、重要な理論的・構造的ポイントについて言及し、shapeless のコードベースからさらなる情報を引き出すための方法を示していく。
