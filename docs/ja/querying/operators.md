---
title: 演算子
sort_rank: 2
---

# 演算子

## 二項演算子

Prometheusのクエリ言語は基本的な論理・算術演算子をサポートしている。
instanceベクトル間の演算に対しては、[マッチングの振る舞い](#vector-matching)を変更することができる。

### 算術二項演算子

Prometheusには以下の二項算術演算子が存在する。

* `+` （加算）
* `-` （減算）
* `*` （乗算）
* `/` （除算）
* `%` （モジュロ）
* `^` （累乗/指数）

二項算術演算子は、scalar/scalar、vector/scalar、vector/vectorの値の組み合わせの間で定義される。

**2つのスカラー間**での振る舞いは明らかである。両方のスカラーのオペランドに演算子を適用した結果のスカラーに評価される。

**instantベクトルとスカラーの間**では、演算子はベクトルのそれぞれの値に適用される。
つまり、時系列のinstantベクトルが2倍されると、結果は、元のベクトルのそれぞれの値が2倍されたベクトルである。

**2つのinstantベクトルの間**では、二項算術演算子は、左辺ベクトルの各要素と右辺ベクトルのそれに[対応する要素](#vector-matching)に適用される。
その結果が結果ベクトルに伝達され、グルーピングラベルが出力されるラベルセットになる。
メトリック名は削除される。
右辺に対応する要素がない要素は、結果ベクトルからもなくなる。

### 比較二項演算子

Prometheusには以下の二項比較演算子が存在する。

* `==` （等しい）
* `!=` （等しくない）
* `>` （より大きい）
* `<` （より小さい）
* `>=` （より大きいまたは等しい）
* `<=` （より小さいまたは等しい）

比較演算子は、scalar/scalar、vector/scalar、vector/vectorの値の組み合わせの間で定義される。
デフォルトで、これらはフィルターである。
これらの振る舞いは、演算子の後に`bool`を与えることで、フィルタリングではなく、`0`または`1`を返すように変更出来る。

**2つのスカラー間**では、`bool`修飾子が与えられる必要があり、演算子の結果は、比較の結果に基づいて`0`(`false`)または`1`(`true`)のスカラーとなる。

**instantベクトルとスカラーの間**では、演算子はベクトルのそれぞれの値に適用される。
比較結果が`false`となるベクトルの要素は結果ベクトルから削除される。
`bool`修飾子が与えられると、削除されるはずだったベクトル要素は`0`、保持されるはずだったベクトル要素は`1`の値を持つようになる。

**2つのinstantベクトルの間**では、デフォルトで、対応する要素に適用されたフィルターとして振る舞う。
結果がtrueでないまたは他方の辺に対応する要素がない要素は、結果から削除される。
その他の要素は、結果ベクトルに伝達され、グルーピングラベルが出力されるラベルセットになる。
`bool`修飾子が与えられると、削除されるはずだったベクトル要素は`0`、保持されるはずだったベクトル要素は`1`の値を持つようになり、グルーピングラベルが出力されるラベルセットになる。

### 論理/集合二項演算子

これらの論理/集合二項演算子は、instantベクトル間にのみ定義される。

* `and` （交差）
* `or` （和）
* `unless` （complement）

`vector1 and vector2`は、`vector2`にラベル集合が完全にマッチする要素がある`vector1`の要素からなるベクトルになる。
その他の要素は削除される。
メトリック名と値は、左辺ベクトルから引き継がれる。

`vector1 or vector2`は、`vector1`の全ての要素（ラベル集合＋値）および`vector1`に対応するラベル集合のない全ての`vector2`の要素からなるベクトルになる。

`vector1 unless vector2`は、ラベル集合が完全にマッチする要素が`vector2`にない`vector1`の要素からなるベクトルになる。
両辺のベクトルでマッチする全ての要素は削除される。

## ベクトルマッチング

ベクトル間の演算は、左辺の要素それぞれに対して、右辺ベクトルから対応する要素を見つけようとする。
対応させる（matching）振る舞いには2つの基本的な種類（1対1および多対1/1対多）がある。

### 1対1マッチング

**1対1マッチング**では、演算子の両辺からユニークなペアを取得する。
デフォルトケースでは、`vector1 <operator> vector2`という形式の演算である。
2つのエントリがマッチングされるのは、完全に同じラベル集合およびその値を持つ場合である。
これに対して、`ignoring`キーワードによって、マッチングする時に特定のラベルを無視することができる。
また、`on`キーワードによって、考慮するラベルを減らすことができる。

    <vector expr> <bin-op> ignoring(<label list>) <vector expr>
    <vector expr> <bin-op> on(<label list>) <vector expr>

入力例:

    method_code:http_errors:rate5m{method="get", code="500"}  24
    method_code:http_errors:rate5m{method="get", code="404"}  30
    method_code:http_errors:rate5m{method="put", code="501"}  3
    method_code:http_errors:rate5m{method="post", code="500"} 6
    method_code:http_errors:rate5m{method="post", code="404"} 21

    method:http_requests:rate5m{method="get"}  600
    method:http_requests:rate5m{method="del"}  34
    method:http_requests:rate5m{method="post"} 120

クエリ例:

    method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m

これは、直近5分間における各メソッドに対するステータスが500となったHTTPリクエストの割合を返す。
仮に`ignoring(code)`を付けないと、ラベル集合が完全に同じメトリクスはなくなってしまう。
また、`put`と`del`メソッドは、マッチするものがないので結果には含まれない。

    {method="get"}  0.04            //  24 / 600
    {method="post"} 0.05            //   6 / 120

### 多対1/1対多マッチング

**多対1/1対多マッチング**とは、1の側のベクトル要素それぞれが多の側の複数の要素と対応するような場合のことをいう。
片側の要素を逆側の複数の要素とマッチングさせるには、明示的に`group_left`または`group_right`を利用する必要がある。
left/rightによって、左右どちらを複数要素にマッチングさせるかが決まる。

    <vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
    <vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
    <vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
    <vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>

グループ修飾子と共に与えられるラベルリストは、結果のメトリクスに含まれるべきラベルが含まれる。
`on`を使う場合、ラベルはどちらか一方のラベルリストにのみ含むことができる。
結果のベクトルに含まれる各時系列は、ユニークに特定されなければならない。

_グルーピング修飾子が使えるのは、[比較](#comparison-binary-operators)と[算術](#arithmetic-binary-operators)だけである。`and`および`unless`、`or`演算子は、デフォルトでは、右辺のvectorの全ての可能な要素にマッチする。_

クエリ例:

    method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m

この例では左辺のベクトルには、`method`ラベルの一つの値に対して複数のエントリーがあるので、`group_left`を使う。
右辺の要素は、同じ`method`ラベルの値を持つ左辺の複数の要素にマッチングされる。

    {method="get", code="500"}  0.04            //  24 / 600
    {method="get", code="404"}  0.05            //  30 / 600
    {method="post", code="500"} 0.05            //   6 / 120
    {method="post", code="404"} 0.175           //  21 / 120

_多対1/1対多マッチングは高度なユースケースであり、慎重に使うべきである。多くの場合、`ignoring(<labels>)`で所望の結果が得られる。_

## 集約演算子

Prometheusは以下のビルトイン集約演算子をサポートする。これらの演算子で、一つのinstance vectorの要素を集約し、集約されて少なくなった要素からなる新しいベクトルを得ることができる。

* `sum` (calculate sum over dimensions)
* `min` (select minimum over dimensions)
* `max` (select maximum over dimensions)
* `avg` (calculate the average over dimensions)
* `stddev` (calculate population standard deviation over dimensions)
* `stdvar` (calculate population standard variance over dimensions)
* `count` (count number of elements in the vector)
* `count_values` (count number of elements with the same value)
* `bottomk` (smallest k elements by sample value)
* `topk` (largest k elements by sample value)
* `quantile` (calculate φ-quantile (0 ≤ φ ≤ 1) over dimensions)

これらの演算子は、**全て**のラベルにまがたって集約することもできるし、`without`や`by`を含めることで個別のラベルを残すこともできる。

    <aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]

パラメーターが必要なのは、`count_values`、`quantile`、`topk`、`bottomk`だけである。
`without`は、結果のベクトルから`label list`に含まれるラベルを取り除き、他の全てのラベルを結果のベクトルに残す。
`by`は、`without`とは逆に、`label list`に含まれないラベルを（ラベルの値が全ての要素で同一だとしても）取り除く。

`count_values`は、ユニークなサンプル値ごとに1つの時系列を出力する。
それぞれの時系列には、ラベルが1つ追加される。
そのラベル名は、集約するパラメーターで与えられ、ラベル値はユニークなサンプル値である。
それぞれの時系列の値は、そのサンプル値が現れた回数である。

`topk`と`bottomk`は、他の集約演算子と違って、入力に含まれたラベルを含んだ入力ベクトルの部分集合が結果ベクトルとして返される。
`by`と`without`は、入力ベクトルをどうまとめるか表すためだけに利用される。

例：

メトリック`http_requests_total`がラベル`application`、`instance`、`group`の値を複数持つ時系列である場合、`application`と`group`ごとの、全ての`instances`についての合計HTTPリクエスト数を計算は、以下の式でできるだろう。

    sum(http_requests_total) without (instance)

これは、以下の式と同じことである。

     sum(http_requests_total) by (application, group)

**全て**の`application`の合計HTTPリクエスト数に興味があるだけなら、単に以下のように書けば良いだろう。

    sum(http_requests_total)

ビルドバージョンごとのバイナリの数を数えるには、以下のように書けば良いだろう。

    count_values("version", build_version)

全てのインスタンスの中でHTTPリクエスト数の大きいもの5つを得るためには、以下のように書けば良いだろう。

    topk(5, http_requests_total)

## 二項演算子の優先順位

以下のリストで二項演算子の高い方から低い方への優先順位を示す。

1. `^`
2. `*`, `/`, `%`
3. `+`, `-`
4. `==`, `!=`, `<=`, `<`, `>=`, `>`
5. `and`, `unless`
6. `or`

同じ優先順位の演算子は、左結合である。
例えば、`2 * 3 % 2`は、`(2 * 3) % 2`と同じである。
ただし、`^`は、右結合なので、`2 ^ 3 ^ 2`は、`2 ^ (3 ^ 2)`と同じである。
