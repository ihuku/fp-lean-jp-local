# 末尾再帰（Tail Recursion）

Leanの`do`記法を使用すれば、`for`や`while`などの従来のループ構文を使用することができますが、これらの構造は裏側で再帰関数の呼び出しに変換されます。
ほとんどのプログラミング言語では、再帰関数はループに比べて重要なデメリットを持っています：ループはスタック上でスペースを消費しませんが、再帰関数は再帰呼び出しの数に比例してスタックスペースを消費します。
スタックスペースは通常制限されており、再帰関数として自然に表現されるアルゴリズムをループと明示的な可変ヒープに割り当てられたスタックと組み合わせて書き直す必要があります。

関数型プログラミングでは、通常は逆のことが真です。
自然に可変ループとして表現されるプログラムはスタックスペースを消費する場合があり、それらを再帰関数に書き直すと速く実行されることがあります。
これは関数型プログラミング言語の重要な側面の1つ、_末尾呼び出しの除去_によるものです。
末尾呼び出しは、1つの関数から別の関数への呼び出しであり、通常のジャンプにコンパイルでき、現在のスタックフレームを置き換える代わりに新しいスタックフレームをプッシュしないプロセスです。末尾呼び出しの除去は、この変換を実装するプロセスです。

末尾呼び出しの除去は単なるオプションの最適化ではありません。
その存在は効率的な関数型コードを書くための基本的な部分です。
それが役立つためには信頼性が必要です。
プログラマは末尾呼び出しを確実に識別できなければならず、コンパイラがそれらを除去することを信頼できなければなりません。

関数`NonTail.sum`は`Nat`のリストの内容を足し合わせます：
```lean
{{#example_decl Examples/ProgramsProofs/TCO.lean NonTailSum}}
```
この関数をリスト `[1, 2, 3]` に適用すると、以下の評価ステップのシーケンスが生成されます：
```lean
{{#example_eval Examples/ProgramsProofs/TCO.lean NonTailSumOneTwoThree}}
```
評価ステップでは、括弧は`NonTail.sum`への再帰呼び出しを示しています。
言い換えれば、これらの3つの数を足し合わせるには、まずリストが空でないことを確認する必要があります。
リストの先頭（`1`）をリストの残りの部分の合計に加えるために、まずリストの残りの部分の合計を計算する必要があります：
```lean
{{#example_eval Examples/ProgramsProofs/TCO.lean NonTailSumOneTwoThree 1}}
```
しかし、リストの残りの部分の合計を計算するには、プログラムはその部分が空かどうかを確認する必要があります。
部分が空でないことを確認します - 残りの部分は`2`が先頭にあるリストです。
結果として、`NonTail.sum [3]` の返り値を待つステップが発生します：
```lean
{{#example_eval Examples/ProgramsProofs/TCO.lean NonTailSumOneTwoThree 2}}
```
ランタイムコールスタックの目的は、値`1`、`2`、および`3`とそれらを結果に追加する命令を追跡することです。
再帰呼び出しが完了すると、呼び出しを行ったスタックフレームに制御が戻り、各加算のステップが実行されます。
リストの先頭を格納し、それらを追加する命令を格納することは無制限ではありません。それはリストの長さに比例したメモリ空間を必要とします。

関数 `Tail.sum` も `Nat` のリストの内容を足し合わせます：
```lean
{{#example_decl Examples/ProgramsProofs/TCO.lean TailSum}}
```
これをリスト `[1, 2, 3]` に適用すると、以下の評価ステップのシーケンスが生成されます：
```lean
{{#example_eval Examples/ProgramsProofs/TCO.lean TailSumOneTwoThree}}
```
内部のヘルパー関数は再帰的に自身を呼び出しますが、最終結果を計算するために何も覚えておかなくても済む方法で行います。
`Tail.sumHelper` がベースケースに達すると、中間呼び出しは単に再帰呼び出しの結果を変更せずに返すため、制御は直接 `Tail.sum` に戻せます。
言い換えれば、`Tail.sumHelper` の各再帰呼び出しに対して単一のスタックフレームを再利用できます。
末尾呼び出しの除去はスタックフレームの再利用そのものであり、`Tail.sumHelper` は_末尾再帰関数_と呼ばれます。

`Tail.sumHelper` の最初の引数には、通常はコールスタックで追跡する必要がある情報がすべて含まれています。つまり、これまでに出会った数値の合計です。
各再帰呼び出しでは、この引数に新しい情報が更新され、コールスタックに新しい情報を追加するのではなく、置き換えられます。
コールスタックからの情報を置き換えるような引数（`soFar`のような）は_アキュムレータ_と呼ばれます。

執筆時点と筆者のコンピューターでは、`NonTail.sum` は要素数が 216,856 以上のリストを渡すとスタックオーバーフローが発生します。
一方、`Tail.sum` はスタックオーバーフローなしで 100,000,000 の要素からなるリストを合計することができます。
`Tail.sum` を実行中に新しいスタックフレームをプッシュする必要がないため、それは現在のリストを保持する可変変数を持つ`while`ループと完全に等価です。
各再帰呼び出しで、スタック上の関数引数は単にリストの次のノードに置き換えられます。


## 末尾と非末尾の位置（Tail and Non-Tail Positions）

`Tail.sumHelper` が末尾再帰である理由は、再帰呼び出しが_末尾位置_にあるためです。
簡単に言うと、関数呼び出しは、呼び出し元が返された値をどのように変更する必要はなく、そのまま返すだけの場合、末尾位置にあります。
もっと形式的には、末尾位置は式に明示的に定義できます。

`match`式が末尾位置にある場合、その各ブランチも末尾位置にあります。
`match`がブランチを選択した後、制御はすぐにそのブランチに進みます。
同様に、`if`式の両方のブランチは、`if`式自体が末尾位置にある場合、末尾位置にあります。
最後に、`let`式が末尾位置にある場合、その本体も末尾位置にあります。

それ以外の位置は末尾位置にはありません。
関数またはコンストラクタの引数は末尾位置にありません。なぜなら評価は引数の値が適用される関数またはコンストラクタを追跡しなければならないからです。
内部関数の本体も末尾位置にありません。なぜなら制御がそれに移るかもしれないからです：関数本体は関数が呼び出されるまで評価されません。
同様に、関数型の本体も末尾位置にはありません。
`(x : α) → E` で `E` を評価するには、結果の型には `(x : α) → ...` が包含されている必要があります。

`NonTail.sum` では、再帰呼び出しは末尾位置になっていません。なぜなら、それが`+`の引数であるからです。
`Tail.sumHelper` では、再帰呼び出しは末尾位置にあるのは、それ自体が関数の本体であるパターンマッチの直下にあるからです。

執筆時点では、Leanは再帰関数内の直接の末尾呼び出しのみを除去します。
これは、`f` の定義内の `f` への末尾呼び出しは除去されますが、他の関数 `g` への末尾呼び出しは除去されません。
他の関数への末尾呼び出しを除去してスタックフレームを節約することは確かに可能ですが、これはまだLeanで実装されていません。

## リストの反転（Reversing Lists）

関数 `NonTail.reverse` は各サブリストの先頭を結果の末尾に追加することでリストを反転させます：
```lean
{{#example_decl Examples/ProgramsProofs/TCO.lean NonTailReverse}}
```
これを使って `[1, 2, 3]` を逆転させると、次の手順のシーケンスが得られます：
```lean
{{#example_eval Examples/ProgramsProofs/TCO.lean NonTailReverseSteps}}
```

末尾再帰版では、各ステップでアキュムレータに `x :: ·` を使用します。これは、`NonTail.reverse` と計算する際に各スタックフレームに保存されるコンテキストがベースケースから適用されるためです。
各「覚えられた」コンテキストの一部は後入れ先出しの順序で実行されます。
一方、アキュムレータを渡すバージョンでは、リストの最初のエントリからではなく、元のベースケースからではなく、アキュムレータを修正します。これは次の簡約ステップのシリーズで見られます：
```lean
{{#example_eval Examples/ProgramsProofs/TCO.lean TailReverseSteps}}
```

言い換えれば、非末尾再帰版はベースケースから始まり、リストを右から左に反転させるために再帰の結果を修正します。
リスト内のエントリは、アキュムレータに対して最初に来たものから最初に影響を与えます。
アキュムレータを持つ末尾再帰版は、リストの先頭から始まり、リストを左から右に反転させるための初期アキュムレータの値を変更します。

加法は可換であるため、`Tail.sum` にこれに対処するための何も行う必要はありませんでした。
一方、リストの追加は可換ではないため、逆の方向で実行した場合に同じ効果を持つ操作を見つけるために注意が必要です。
`NonTail.reverse` で再帰の結果の後に `[x]` を追加することは、結果が逆順で構築される場合にリストの先頭に `x` を追加するのと同様です。


## 複数の再帰呼び出し（Multiple Recursive Calls）

`BinTree.mirror` の定義では、2つの再帰呼び出しがあります：
```lean
{{#example_decl Examples/Monads/Conveniences.lean mirrorNew}}
```
命令型言語が通常、`reverse` や `sum` のような関数に対して while ループを使用するのと同様に、この種のトラバーサルに対して再帰関数を通常使用します。
この関数は、アキュムレータを渡すスタイルを使用して簡単に末尾再帰に書き直すことはできません。

通常、再帰的なステップごとに複数の再帰呼び出しが必要な場合、アキュムレータを渡すスタイルを使用することは難しいでしょう。
この難しさは、再帰関数をループと明示的なデータ構造を使用するように書き直す難しさに似ており、関数が終了することをLeanに納得させる複雑さが追加されています。
しかし、`BinTree.mirror` のように、複数の再帰呼び出しはしばしば、それ自体の複数の再帰的な出現を持つコンストラクタを持つデータ構造を示します。
これらの場合、構造の深さは通常全体のサイズに対して対数的であり、スタックとヒープのトレードオフがはるかに緩和されます。
これらの関数を末尾再帰にするための体系的なテクニックがあります。例えば、_継続渡しスタイル_ を使用することができますが、それはこの章のスコープ外です。


## 演習（Exercises）

次の非末尾再帰関数をアキュムレータを渡す末尾再帰関数に翻訳してください：

```lean
{{#example_decl Examples/ProgramsProofs/TCO.lean NonTailLength}} 
```

```lean
{{#example_decl Examples/ProgramsProofs/TCO.lean NonTailFact}}
```

`NonTail.filter` の翻訳は、入力リストの長さに対して定数のスタックスペースを取り、入力に対する時間が線形にかかります。
オリジナルに対して定数のオーバーヘッドは許容されます：
```lean
{{#example_decl Examples/ProgramsProofs/TCO.lean NonTailFilter}}
```
