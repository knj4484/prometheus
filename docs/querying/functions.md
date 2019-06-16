---
title: クエリ関数
nav_title: 関数
sort_rank: 3
---

# 関数

いくつかの関数にはデフォルト引数がある。
例えば`year(v=vector(time()) instant-vector)`は、1つの引数`v`があり、もしこれが与えられていなければデフォルト値`vector(time())`になるという意味である。

## `abs()`

`abs(v instant-vector)`は、全てのサンプル値をそれらの絶対値に変換された入力ベクトルを返す

## `absent()`

`absent(v instant-vector)`は、

* 渡されたベクトルに何らかの要素があれば、空のベクトルを返す
* 渡されたベクトルに何も要素がなければ、値1の1要素のベクトルを返す

これは、与えられたメトリック名とラベルの組み合わせに対応する時系列データがない場合のアラートのために便利である。

```
absent(nonexistent{job="myjob"})
# => {job="myjob"}

absent(nonexistent{job="myjob",instance=~".*"})
# => {job="myjob"}

absent(sum(nonexistent{job="myjob"}))
# => {}
```

二つ目の例で、`absent()`は、入力ベクトルから1要素の出力ベクトルのラベルを賢く取り出そうとしている。

## `ceil()`

`ceil(v instant-vector)`は、`v`の全要素のサンプル値を一番近い整数に切り上げて丸める。

## `changes()`

`changes(v range-vector)`は、入力の各時系列に対してその値が指定された時間幅内に変化した回数をinstant vectorとして返す。

## `clamp_max()`

`clamp_max(v instant-vector, max scalar)`は、`v`の全要素のサンプル値を上限`max`までにする。

## `clamp_min()`

`clamp_min(v instant-vector, min scalar)`は、`v`の全要素のサンプル値を下限`max`までにする。

## `day_of_month()`

`day_of_month(v=vector(time()) instant-vector)`は、与えられたUTCの時間それぞれに対して日にちを返す。
返される値は、1から31までである。

## `day_of_week()`

`day_of_week(v=vector(time()) instant-vector)`は、与えられたUTCの時間それぞれに対して曜日を返す。
返される値は、0から6までである。
0は日曜を表す。

## `days_in_month()`

`days_in_month(v=vector(time())`は、与えられたUTCの時間それぞれに対してその月の日数を返す。
返される値は、28から31までである。

## `delta()`

`delta(v range-vector)`は、入力ベクトル`v`の各時系列の最初と最後の値の差分を計算し、その差分に対応するラベルのついたinstantベクトル返す。
差分は、range vectorセレクターで指定された時間幅全体をカバーするために外挿される。
したがって、サンプル値が全て整数でも結果が整数でないことがあり得る。

以下の例は、CPUの温度の現在と2時間前の差を返す。

```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

`delta`はゲージとのみ使うべきである。

## `deriv()`

`deriv(v range-vector)`は、[simple linear regression](https://en.wikipedia.org/wiki/Simple_linear_regression)を用いて、入力ベクトル`v`の時系列の秒間の微分を計算する。

`deriv`はゲージとのみ使うべきである。

## `exp()`

`exp(v instant-vector)`は、`v`の全要素に対して指数関数を計算する。特殊ケースは、以下の通り。

* `Exp(+Inf) = +Inf`
* `Exp(NaN) = NaN`

## `floor()`

`floor(v instant-vector)`は、`v`の全要素のサンプル値を一番近い整数に切り捨てて丸める。

## `histogram_quantile()`

`histogram_quantile(φ float, b instant-vector)`は、ヒストグラムのバケット`b`からφ分位数(0 ≤ φ ≤ 1)を計算する
（φ分位数とヒストグラムの一般的な使い方の詳細な説明は[ヒストグラムとサマリー](https://prometheus.io/docs/practices/histograms)を参照すること）。`b`の値は、各バケットに入る観測値の数である。
各バケットには、バケットの上限を表す値を持つラベル`le`がなければならない（そういうラベルがない値は無視される）。
[メトリック型ヒストグラム](https://prometheus.io/docs/concepts/metric_types/#histogram)は、`_bucket`をサフィックスに持つ時系列と適切なラベルを提供する。

分位数の計算のためのタイムウインドウを指定するために関数`rate()`を利用すること。

例として、ヒストグラムのメトリックが`http_request_duration_seconds`だとする。
過去10分間のリクエスト持続時間の90パーセンタイルを計算するために以下の式を用いる。

    histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[10m]))

分位数は、`http_request_duration_seconds`のラベルの組み合わせそれぞれに対して計算される。
集約するためには、`sum()`を使って`rate()`の結果を囲む。
`le`ラベルは、`histogram_quantile()`に必要なので、`by`の中に含めなければならない。
以下の式は、90パーセンタイルを`job`ごとに集約する。

    histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (job, le))

全て集約するためには、`le`ラベルのみを指定する。

    histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (le))

 `histogram_quantile()`は、バケット内が線形に分布していると仮定して分位数の値を補間する。
一番大きいバケットは、上限が`+Inf`でなければならない（そうでなければ、`NaN`が返される）。
分位数が一番大きいバケットに位置している場合、2番目に大きいバケットの上限が返される。
一番小さいバケットの下限は、そのバケットの上限が0より大きければ、0と仮定される。
その場合、そのバケット内では通常の線形補間が適用される。
そうでない場合、一番小さいバケットに位置している分位数に対して一番小さいバケットの上限が返される。

`b`のバケットが2つより少ない場合、`NaN`が返される。
φ < 0に対しては、`-Inf`が返される。
φ > 1に対しては、`+Inf`が返される。

## `holt_winters()`

`holt_winters(v range-vector, sf scalar, tf scalar)`は、`v`のレンジに基づいて時系列の平滑化された値を生成する。
`st`(smoothing factor)が小さいほど、古いデータにより重きが置かれる。
`tf`(trend factor)が大きいほど、データのトレンドがより考慮される。
`sf`と`tf`はどちらも0と1の間でなければならない。

`holt_winters`は、ゲージとのみ利用すべきである。

## `hour()`

`hour(v=vector(time())`は、与えられたUTCの時間それぞれに対して何時かを返す。
返される値は、0から23までである。

## `idelta()`

`idelta(v range-vector)`は、`v`の最後の2つのサンプルの差を計算し、その差分および同じラベルを持つinstant vectorを返す。

`idelta`は、ゲージとのみ利用すべきである。

## `increase()`

`increase(v range-vector)`は、range vector中の時系列の増加を計算する。
単調性が分割されている場合（監視対象の再起動でカウンターがリセットされた場合など）は自動的に補正される。
増分は、指定された時間幅全体をカバーするために外挿法によって推定されるので、カウンターの増加が整数だったとしても、整数でない結果を得ることありうる。

次の例は、最新5分間で計測されたHTTPリクエスト数を返す。

```
increase(http_requests_total{job="api-server"}[5m])
```

`increase`は、カウンターと共にだけ利用するべきである。
`increase(v)`は`rate(v)`に時間幅内の秒数をかけたもののsyntactic sugarであり、主に人間の読みやすさのために用いられるべきである。
レコーディングルールでは、増加が一貫して1秒あたりを基準に追跡されるように、`rate`を用いること。

## `irate()`

`irate(v range-vector)`は、入力ベクトルの時系列の1秒あたりの瞬間的な増加率を計算する。
これは、最後の2つのデータポイントに基づいて計算される。
単調性が分割されている場合（監視対象の再起動でカウンターがリセットされた場合など）は自動的に補正される。

以下のサンプルは、入力ベクトルの時系列それぞれについて、一番最近の2つのデータポイントを5分過去まで探して、HTTPリクエストの秒間レートを返す。

```
irate(http_requests_total{job="api-server"}[5m])
```

`irate`は、不安定で動きが速いカウンターをグラフ化するときにのみ利用すべきである。
rateでの短期の変化は`FOR`をリセットしてしまい、完全に稀なスパイクからなるグラフは読みにくいので、
アラートと動きの遅いカウンターには`rate`を利用すること。

[集約演算子](operators.md#aggregation-operators)（例えば`sum()`）や時間に対して集約する関数（`_over_time`で終わる関数全て）と一緒に`irate()`を使う時には、`irate()`を必ず最初にとり、その後集約するように注意すること。
そうしないと、監視対象が再起動した時のカウンターのリセットを`irate()`が検知できない。

## `label_join()`

`label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)`は、`v`の各時系列に対して、全ての`src_labels`の全ての値を、`separator`を使って連結し、`dst_label`が連結した値を含む時系列を返す。
この関数では、任意の数の`src_labels`が存在して良い。

この例では、各時系列がラベル`foo`に値`a,b,c`を付与されたベクトルを返す。

```
label_join(up{job="api-server",src1="a",src2="b",src3="c"}, "foo", ",", "src1", "src2", "src3")
```

## `label_replace()`

`label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)`は、vの各時系列に対して、正規表現`regex`とラベル`src_label`をマッチングする。もし、マッチしたら、その時系列のラベル`dst_label`が`replacement`に展開されて返される。
`$1`は1つ目、`$2`は2つ目、`$3`以降も同様のマッチしたサブグループで置換される。
もし正規表現がマッチしなければ、時系列が変更されずに返される。

この例では、各時系列がラベル`foo`に値`a`を付与されたベクトルを返す。

```
label_replace(up{job="api-server",service="a:c"}, "foo", "$1", "service", "(.*):.*")
```

## `ln()`

`ln(v instant-vector)`は、`v`の全要素の自然対数を計算する。
特殊ケースは、以下の通り。

* `ln(+Inf) = +Inf`
* `ln(0) = -Inf`
* `ln(x < 0) = NaN`
* `ln(NaN) = NaN`

## `log2()`

`log2(v instant-vector)`は、`v`の全要素の２を底とする対数を計算する。
特殊ケースは、`ln`と同じである。

## `log10()`

`log10(v instant-vector)`は、`v`の全要素の10を底とする対数を計算する。特殊ケースは`ln`と同じである。

## `minute()`

`minute(v=vector(time())`は、与えられたUTCの時間それぞれの分を返す。
返される値は、0から59である。

## `month()`

`month(v=vector(time())`は、与えられたUTCの時間それぞれの月を返す。
返される値は、1から12である（1が1月を意味する）。

## `predict_linear()`

`predict_linear(v range-vector, t scalar)`は、[simple linear regression](https://en.wikipedia.org/wiki/Simple_linear_regression)を用いて、今から`t`秒後の値をrange vector `v`に基づいて予測する。

`predict_linear`は、ゲージと共にのみ利用すべきである。

## `rate()`

`rate(v range-vector)`は、入力range vectorの時系列の1秒あたりの増加率を計算する。
単調性が分割されている場合（監視対象の再起動でカウンターがリセットされた場合など）は自動的に補正される。
また、スクレイプが失敗していたりスクレイプ周期が完全には整列していなくてもいいように、時系列の末尾を外挿法によって推定する。

次の例は、range vector内の時系列ごとの最新5分間に計測されたHTTPリクエストの1秒あたりの増加率を返す。

```
rate(http_requests_total{job="api-server"}[5m])
```

`rate`は、カウンターと共にだけ利用するべきである。
アラートや動きの遅いカウンターのグラフ化に最も適している。

[集約演算子](operators.md#aggregation-operators)（例えば`sum()`）や時間に対して集約する関数（`_over_time`で終わる関数全て）と一緒に`rate()`を使う時には、`rate()`を必ず最初にとり、その後集約するように注意すること。
そうしないと、監視対象が再起動した時のカウンターのリセットを`rate()`が検知できない。

## `resets()`

`resets(v range-vector)`は、入力された時系列それぞれに対して、与えられた時間幅でのリセットの回数をinstantベクトルとして返す。
2つの連続するサンプル間で減少があれば、カウンターのリセットとみなされる。

`resets`は、カウンターとのみ利用すべきである。

## `round()`

`round(v instant-vector, to_nearest=1 scalar)`は、`v`の全要素のサンプル値を一番近い整数へ丸める。
オプション引数`to_nearest`によって、サンプル値を何の倍数に丸めるかを指定できる。
この倍数は、分数でも良い。

## `scalar()`

`scalar(v instant-vector)`は、要素が1つのベクトルを受け取り、その1つの要素をスカラーとして返す。
入力ベクトルの要素がちょうど1つでない場合、`NaN`が返される。

## `sort()`

`sort(v instant-vector)`は、それらのサンプル値を昇順でソートされたベクトル要素を返す。

## `sort_desc()`

`sort`と同様だが、降順でソートする。

## `sqrt()`

`sqrt(v instant-vector)`は、`v`の全要素の平方根を計算する。

## `time()`

`time()`は、UTCの1月1日からの秒数を返す。
これは、実際には、現在時刻ではなく、式が評価された時刻を返すことに注意すること。

## `timestamp()`

`timestamp(v instant-vector)`は、与えられたベクトルの各サンプルのタイムスタンプをUTCの1970年1月1日からの秒数として返す。

*この関数は、Prometheus 2.0で追加された。*

## `vector()`

`vector(s scalar)`は、スカラー`s`をラベルのないベクトルとして返す。

## `year()`

`year(v=vector(time()) instant-vector)`は、与えられたUTCの各時間の年を返す。

## `<aggregation>_over_time()`

以下の関数によって、与えられた`range-vector`それぞれを時間に対して集約し、時系列ごとの結果を含むinstant vectorを返すことができる。

* `avg_over_time(range-vector)`: 指定されたインターバル中の全ての点の平均値
* `min_over_time(range-vector)`: 指定されたインターバル中の全ての点の最小値
* `max_over_time(range-vector)`: 指定されたインターバル中の全ての点の最大値
* `sum_over_time(range-vector)`: 指定されたインターバルの全ての値の和
* `count_over_time(range-vector)`: 指定されたインターバルの全ての値の数
* `quantile_over_time(scalar, range-vector)`:指定されたインターバルの全ての値のφ-quantile (0 ≤ φ ≤ 1)
* `stddev_over_time(range-vector)`: 指定されたインターバルの値のpopulation standard deviation（母標準偏差）
* `stdvar_over_time(range-vector)`: 指定されたインターバルの値のpopulation standard variance

値がインターバル全体で等間隔でない場合でも、指定されたインターバルの全ての値が集約で同じ重みを持つことに注意すること。
