---
title: クエリの基礎
nav_title: 基礎
sort_rank: 1
---

# <span class="anchor-text-supplement">Querying </span>Prometheusのクエリ

Prometheusは、ユーザーがリアルタイムに時系列データを抽出したり集約出来るような言語を提供している。
ある式の結果は、グラフに表示したり、Prometheusのexpression browserで表形式のデータとして見たり、[HTTP API](api.md)経由で外部システムに利用させることが出来る。

## <span class="anchor-text-supplement">Examples</span>利用例

このページはリファレンスとして意図されている。
学習には、いくつかの[利用例](examples.md)から始めるのが簡単であろう。

## <span class="anchor-text-supplement">Expression language data types</span>データ型

Prometheusでは、ある式またはサブ式は、評価されて四つの型のうちの一つになる。

* **Instant vector** - 同一のタイムスタンプを共有していて、それぞれ一つだけサンプルを含む時系列の集合
* **Range vector** - 時間幅に渡るデータをそれぞれ含んでいる時系列の集合
* **Scalar** - 単純な浮動小数点数の数値
* **String** - 単純な文字列の値（現在は未使用）

ユースケース（式の結果をグラフ化するか表示するか）によって、これらの型の中でユーザーの指定した式の結果として正しいものは限られている。
例えば、直接グラフ化出来るのは、instant vectorを返す式のみである。

## <span class="anchor-text-supplement">Literals</span>リテラル

### <span class="anchor-text-supplement">String literals</span>文字列リテラル

文字列は、シングルクオート、ダブルクオート、バッククオートで挟むことでリテラルとして指定出来る。

PromQLは、[Goのエスケープ](https://golang.org/ref/spec#String_literals)と同じルールに従う。
シングルクオートとダブルクオート内では、バックスラッシュがエスケープシーケンスを開始し、`a`、`b`、`f`、`n`、`r`、`t`、`v`、`\`を次に書くことが出来る。
8進数表記(`\nnn`)または16進数表記(`\xnn`、`\unnnn`、`\Unnnnnnnn`)を使うことで特殊文字を表すことが出来る。

バッククオート内では、エスケープは処理されない。Goと違い、Prometheusはバッククオート内の改行を取り除かない。

例

    "this is a string"
    'these are unescaped: \n \\ \t'
    `these are not unescaped: \n ' " \t`

### <span class="anchor-text-supplement">Float literals</span>浮動小数点数リテラル

スカラーのfloat値は、`[-](digits)[.(digits)]`の形式の数値として文字通り記述出来る。

    -2.43

## <span class="anchor-text-supplement">Time series Selectors</span>時系列セレクター

### instant vector<span class="anchor-text-supplement"> selectors</span>セレクター

instant vectorセレクターによって、時系列の集合とその各時系列の与えられたタイムスタンプの一つのサンプル値を抽出することが出来る。
一番簡単な形として、メトリック名だけを指定する。
その結果は、このメトリック名を持つ時系列全ての要素を含むinstant vectorになる。

この例では、`http_requests_total`というメトリック名の全ての時系列を抽出する。

    http_requests_total

マッチさせるラベルの集合を`{}`内に追加することで、時系列をさらにフィルタリングすることが可能である。

この例では、`http_requests_total`というメトリック名を持ちかつラベル`job`が`prometheus`に、ラベル`group`が`canary`にセットされている時系列のみが抽出される。

    http_requests_total{job="prometheus",group="canary"}

ラベル値に対する否定的なマッチングや正規表現マッチングも可能である。
以下のラベルマッチング演算子が存在する。

* `=`: 与えられた文字列と完全に等しいラベルを抽出する
* `!=`: 与えられたラベルに等しくないラベルを抽出する
* `=~`: 与えられた文字列に正規表現がマッチするラベルを抽出する
* `!~`: 与えられた文字列に正規表現がマッチしないラベルを抽出する

例えば、次の式は、`staging`または`testing`、`development`環境に対する`GET`以外のHTTPメソッドの全ての時系列`http_requests_total`を抽出する

    http_requests_total{environment=~"staging|testing|development",method!="GET"}

空のラベル値にマッチするラベルマッチャーは、指定されたラベルをそもそも持っていない時系列も抽出する。
正規表現マッチングは、完全にアンカーされている。同じラベル名に対する複数のマッチャーを持つことが可能である。

ベクトルセレクターは、名前を指定するか、空文字列にマッチしないラベルマッチャーを最低一つ指定しなければいけない。
以下の式は不正である。

    {job=~".*"} # Bad!

それに対し、これらの式は、空のラベル値にマッチしないセレクターを持っているので、正当である。

    {job=~".+"}              # Good!
    {job=~".*",method="get"} # Good!

ラベルマッチャーは、内部ラベル`__name__`に対してマッチングすることで、メトリック名にも適用できる。
例えば、`http_requests_total`という式は、`{__name__="http_requests_total"}`と等価である。
`=`以外のマッチャー（`!=`、`=~`、`!~`）も利用出来る。
次の式は、`job:`で始まる名前の全てのメトリクスを抽出する。

    {__name__=~"job:.*"}

Prometheusの全ての正規表現は[RE2 syntax](https://github.com/google/re2/wiki/Syntax)を利用する。

### range vector<span class="anchor-text-supplement"> Selectors</span>セレクター

range vectorリテラルは、現時点から過去のある時間幅のサンプルを抽出すること以外は、instant vectorリテラルのように機能する。
時間幅は、vectorセレクター末尾の角括弧(`[]`)内に追加され、結果の各range vectorの要素としてどれぐらいの過去まで戻って値が取得されるべきかを指定する。

時間幅は、数値とそれに続く以下のいずれかの単位として指定される。

* `s` - seconds
* `m` - minutes
* `h` - hours
* `d` - days
* `w` - weeks
* `y` - years

この例では、ラベル`job`が`prometheus`にセットされた`http_requests_total`というメトリクス名の全ての時系列の最新の5分間で記録された値全てを抽出する。

    http_requests_total{job="prometheus"}[5m]

### offset<span class="anchor-text-supplement"> modifier</span>修飾子

`offset`修飾子によって、クエリ中の個々のinstant vectorまたはrange vectorの時間オフセットの変更ができる。

例えば、以下の式は、クエリ評価時から相対的に5分過去の`http_requests_total`の値を返す。

    http_requests_total offset 5m

`offset`修飾子は、常にセレクターの直後でなければならないことに注意。すなわち、以下の式は正しい。

    sum(http_requests_total{method="GET"} offset 5m) // GOOD.

それに対して、次の式は*正しくない*。

    sum(http_requests_total{method="GET"}) offset 5m // INVALID.

range vectorに対しても同様に作用する。以下の式は、`http_requests_total`が一週間前に持っていた5分間のrateを返す。

    rate(http_requests_total[5m] offset 1w)

## <span class="anchor-text-supplement">Subquery</span>サブクエリ

サブクエリによって、与えられた時間幅と解像度で手軽にクエリを実行できる。
サブクエリの結果はrange vectorである。

構文: `<instant_query> '[' <range> ':' [<resolution>] ']' [ offset <duration> ]`

* `<resolution>`はオプション。デフォルトはグローバルの評価間隔。

## <span class="anchor-text-supplement">Operators</span>演算子

Prometheusは、たくさんの二項演算子と集約演算子をサポートしている。
これらは、[expression language operators](operators.md)のページに詳しく記述されている。

## <span class="anchor-text-supplement">Functions</span>関数

Prometheusは、データを操作するいくつかの関数をサポートしている。
これらは、[expression language functions](functions.md)のページに詳しく記載されている。

## Gotchas

### <span class="anchor-text-supplement">Staleness</span>失効

クエリ実行時に、サンプルデータのタイムスタンプは、実際の現在の時系列データから独立して選択される。
これは主に、集約される複数の時系列データが時間軸で正確に揃っていないような集約（`sum`、`avg`など）をするケースをサポートするためである。
そのように時間軸の独立性があるので、Prometheusは、関連する各時系列のタイムスタンプに対して値を割り当てなければならないが、
そのタイムスタンプの直前の値を取ることでその割り当てをしている。

もし、対象のスクレイプやルールの評価が、以前は存在していた時系列のサンプルを返さなくなってしまったら、その時系列は、失効（stale）としてマークされる。
もし、監視対象が削除されたら、以前返されていた時系列はその後しばらくして失効としてマークされる。

ある時系列が失効とマークされた後のタイムスタンプでクエリが評価されると、その時系列に対して何の値も返されない。
もし、その時系列の新しいサンプルが引き続いて取得されれば、それらの時系列は通常の状態に戻る。

サンプリングのタイムスタンプから前の（デフォルトで）5分間にサンプルがなければ、その時点での時系列として何の値も返さない。
これは、実質的に、ある時系列の最新で取得されたサンプルが5分より昔である（staleとマークされた）時点でのグラフからその時系列は消滅することを意味している。

staleであることは、スクレイプにタイムスタンプを含んでいる時系列にはマークされない。
その場合、5分の閾値のみが適用される。

### <span class="anchor-text-supplement">Avoiding slow queries and overloads</span>遅いクエリと過負荷の回避

あるクエリが大量のデータを操作する必要がある場合、そのグラフ化は、タイムアウトしたり、サーバーやブラウザを過負荷にする可能性がある。
したがって、未知のデータに対するクエリを構築する際は常に、結果が適量（最大で数百）になるまで、Prometheusのexpression browserの表形式の表示でのクエリの構築から始めること。
十分にデータのフィルタリングや集約が出来てからのみ、グラフモードに切り替えること。もし、その式がアドホックにグラフ化するには時間を食い過ぎるなら、[レコーディングルール](../configuration/recording_rules.md#recording-rules)でpre-recordすること。

このことは、`api_http_requests_total`のような単純なメトリック名のセレクターが様々なラベルを持つ何千もの時系列に展開され得るPrometheusのクエリ言語にとって特に重要である。
たくさんの時系列を集約する式が、出力がほんの少数の時系列だとしても、サーバーでの負荷を生み出す可能性に注意すること。
これは、RDBでのカラムの値全ての合計が、出力が単なる一つの数値にも関わらず、遅くなりうるのと同様である。
