## まとめ

本章では、shapeless がどのように自然数を表現しているか、およびそれを型クラスでどのように利用しているのかについて説明した。
いくつかの事前定義された演算型クラスによって、長さを求めたり、添字によって要素にアクセスしたりすることができ、また `Nat` を用いて独自の型クラスを作ることもできることを見た。

`Nat`、`Poly`、その他最後の数章で見てきた様々な型は、`shapeless.ops` で提供されている道具箱のほんの一部でしかない。
他にも、包括的なコードの構成要素を提供する、たくさんの演算型クラスがある。
しかし、本書で挙げた理論さえあれば、独自の型クラスを導出するのに必要な大部分の演算を理解するのに十分だといえる。
`shapeless.ops` パッケージのソースコードは、今や他の便利な演算を自分で探し出せるほどには身近なものになったはずだ。
