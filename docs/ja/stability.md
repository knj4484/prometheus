---
title: APIの安定性
sort_rank: 9
---

# API<span class="anchor-text-supplement"> Stability Guarantees</span>の安定性保証

Prometheusは、1つのメジャーバージョン内でのAPIの安定性を約束しており、鍵となる機能の破壊的な変更を避けるように努力する。 ただし、見栄えに関する機能、まだ開発中の機能、サードパーティのサービスに依存する機能には、これが当てはまらないものもある。

2.xで安定と考えられているものは以下の通り。

* クエリ言語とデータモデル
* アラートルールとレコーディングルール
* メトリクスの出力フォーマット
* v1 HTTP API（ダッシュボードとUIで利用される）
* 設定ファイルのフォーマット（サービスディスカバリーとremote read/writeは除く。下記参照）
* ルール/アラートのファイルフォーマット
* コンソールテンプレートの構文と意味

2.xで不安定と考えられているものは以下の通り。

* 以下を含む、実験的または変更を受けるとされている機能
  * [PromQL関数holt_winters](https://github.com/prometheus/prometheus/issues/2458)
  * remote read、remote writeおよびremote readエンドポイント
  * v2 HTTPとGRPC APIs
* サービスディスカバリー連携（`static_configs`と`file_sd_configs`を除く）
* サーバーの一部であるパッケージのGo API
* Web UIで生成されるHTML
* エンドポイント/metricsで出力されるPrometheus自体のメトリクス
* ディスク上のフォーマット。ただし、もし変更があっても前方互換でPrometheusは透過的に扱うだろう

実験的（experimental）/不安定（unstable）と記された機能を使わない限り、あるメジャーバージョン内でのアップグレードは 普通は調整なしで行うことが出来て、何かが壊れるリスクはとても少ない。 破壊的な変更は、リリースノートで`CHANGE`と記される。