---
title: クエリ例
nav_title: 例
sort_rank: 4
---

# クエリ例

## 単純な時系列の選択

`http_requests_total`メトリックの全ての時系列を返す

    http_requests_total

`http_requests_total`メトリックで`job`と`handler`ラベルが指定された値であるものを返す

    http_requests_total{job="apiserver", handler="/api/comments"}

ある時間幅（ここでは5分）の全ての値をrange vectorにして返す

    http_requests_total{job="apiserver", handler="/api/comments"}[5m]

range vectorを返す式は、直接グラフに表示できないが、Consoleで表としてみることができる。

正規表現で`job`がパターンに合う（ここでは`server`で終わる）時系列を選択することができる。

正規表現を利用することで、`job`がパターンに合う（ここでは`server`で終わる）時系列を選択することができる。
文字列全体のマッチではなく部分文字列のマッチを行うことに注意。

    http_requests_total{job=~".*server"}

Prometheusの正規表現は[RE2 syntax](https://github.com/google/re2/wiki/Syntax)を使う。

HTTPステータスが4xx以外のものの選択は以下のようにしてできるだろう。

    http_requests_total{status!~"4.."}

## サブクエリ

このクエリは、メトリック`http_requests_total`の5分間の増加を過去30分に対して1分の解像度で返す。

    rate(http_requests_total[5m])[30m:1m]

これは入れ子になったサブクエリの例である。
関数`deriv`に対するサブクエリは、デフォルトの解像度を用いる。
不必要にサブクエリを利用するのは賢明ではない。

    max_over_time(deriv(rate(distance_covered_total[5s])[30s:5s])[10m:])

## 関数、演算子など

直近5分の`http_requests_total`の1秒あたり増加率を全て返す。

    rate(http_requests_total[5m])

時系列`http_requests_total`には全て`job`および`instance`ラベルがあると前提で、全てのinstanceに渡ってrateを合計したい（つまり、少なくなった時系列を得るが`job`軸を残したい）とする。

    sum(rate(http_requests_total[5m])) by (job)

同じラベルを持つ二つのメトリクスがある場合、二項演算子を使って、同じラベル集合を持つ両辺の要素を演算した結果を得ることができる。
この例では、（これらのメトリクスを出力する架空のクラスタのスケジューラーが稼働させている）各インスタンスの未使用メモリをMiB単位で返す。

    (instance_memory_limit_bytes - instance_memory_usage_bytes) / 1024 / 1024

同じ式をアプリケーション毎に合計したものは次のように書けるだろう。

    sum(
      instance_memory_limit_bytes - instance_memory_usage_bytes
    ) by (app, proc) / 1024 / 1024

各インスタンスに対して`instance_cpu_time_ns`が次のように出力されているとする。

    instance_cpu_time_ns{app="lion", proc="web", rev="34d0f99", env="prod", job="cluster-manager"}
    instance_cpu_time_ns{app="elephant", proc="worker", rev="34d0f99", env="prod", job="cluster-manager"}
    instance_cpu_time_ns{app="turtle", proc="api", rev="4d3a513", env="prod", job="cluster-manager"}
    instance_cpu_time_ns{app="fox", proc="widget", rev="4d3a513", env="prod", job="cluster-manager"}
    ...

CPU利用の多い`app`と`proc`でグループされた上位3つを取得することができる。

    topk(3, sum(rate(instance_cpu_time_ns[5m])) by (app, proc))

このメトリックが稼働中のインスタンスにつき1つの時系列を持っているとすると、アプリケーションごとの稼働中インスタンス数は以下のようにして計算できる。

    count(instance_cpu_time_ns) by (app)
