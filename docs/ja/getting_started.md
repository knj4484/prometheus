---
title: 始め方
sort_rank: 1
---

# <span class="anchor-text-supplement">Getting started</span>始め方

このガイドは、Hello World形式のチュートリアルである。
簡単なサンプルのセットアップを通して、Prometheusをインストールし、設定し、利用する方法を示す。
ローカルで、Prometheusをダウンロードし、実行し、自分自身とサンプルアプリケーションをscrapeするように設定することになる。
そして、収集した時系列を利用するためのクエリ、ルール、グラフに取り組むことになる。

## <span class="anchor-text-supplement">Downloading and running </span>Prometheusのダウンロードと実行

自分のプラットフォームに合った[Prometheusの最新リリース](https://prometheus.io/download)をダウンロードし、展開、実行する

```bash
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

Prometheusを起動する前に、設定をしよう。

## <span class="anchor-text-supplement">Configuring </span>自分自身を監視するPrometheusの設定<span class="anchor-text-supplement"> to monitor itself</span>

Prometheusは、監視対象のHTTPのエンドポイントをスクレイプすることで監視対象からメトリクスを収集する。 Prometheusはまた自分自身に関して同じ方法でデータをexposeしているので、自分自身の状態をスクレイプし、監視することができる。

自分自身のデータしか収集しないPrometheusサーバーは、実際は特に有益ではないが、最初の例としては良い。
以下のPrometheusの基本的な設定を`prometheus.yml`という名前のファイルに保存する。

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

*訳注: external_labelsには、ある組織内のPrometheusはそれぞれユニークな値を付けるべきである。*

設定項目の全ての仕様については、[設定のドキュメント](configuration/configuration.md)を参照すること。

## <span class="anchor-text-supplement">Starting </span>Prometheusの起動

新しく作成した設定ファイルでPrometheusを起動するには、Prometheusのバイナリがあるディレクトリに移動して、以下のコマンドを実行する

```bash
# Start Prometheus.
# By default, Prometheus stores its database in ./data (flag --storage.tsdb.path).
./prometheus --config.file=prometheus.yml
```

Prometheusが起動し、[localhost:9090](http://localhost:9090)でPrometheus自体のステータスページを見ることができるはずである。
Prometheusが自分自身のデータをメトリクスのHTTPエンドポイントから収集するためにしばらく時間を与える。

また、メトリクスのエンドポイント[localhost:9090/metrics](http://localhost:9090/metrics)に行くことで、Prometheusが自分自身のメトリクスを提供していることを確認できる。

## <span class="anchor-text-supplement">Using the </span>expressionブラウザの利用<span class="anchor-text-supplement"> browser</span>

Prometheusが自分自身について収集したデータをいくつか見てみることにしよう。 Prometheusに組み込まれたexpressionブラウザを使うには、 http://localhost:9090/graph に行って、Consoleビューを選択する。

[localhost:9090/metrics](http://localhost:9090/metrics)から収集出来る通り、Prometheusが自分自身について出力しているメトリクスの一つに`prometheus_target_interval_length_seconds`（対象のscrape間の実際の時間）がある。 expressionコンソールに下記を入力する。

```
prometheus_target_interval_length_seconds
```

これは、全て`prometheus_target_interval_length_seconds`という名前でラベルの異なるたくさんの時系列を（それぞれに対する最新の記録された値とともに）返すはずである。 これらのラベルは、異なるレイテンシーのパーセンタイルと監視対象グループのintervalを示している。

99パーセンタイルのレイテンシーのみに興味があるとすると、その情報を取得するために以下のクエリを使うこともできる。

```
prometheus_target_interval_length_seconds{quantile="0.99"}
```

返された時系列の数をカウントするには、以下のように書くことができる。

```
count(prometheus_target_interval_length_seconds)
```

クエリ言語のさらなる情報については、[クエリ言語のドキュメント](querying/basics.md)を参照すること。

## <span class="anchor-text-supplement">Using the graphing interface</span>グラフ化インターフェースの利用

expressionをグラフ化するには、 http://localhost:9090/graph に行ってGraphタブを利用する

例えば、自分自身がscrapeされているPrometheusに作成されているチャンクの秒間レートをグラフ化する以下の式を入力する

```
rate(prometheus_tsdb_head_chunks_created_total[1m])
```

グラフの幅のパラメーターや他の設定を試してみること。

## <span class="anchor-text-supplement">Starting up some sample targets</span>サンプル監視対象の起動

これをもっと面白くし、Prometheusがscrapeするサンプルの監視対象を起動してみよう。

Goクライアントライブラリには、異なる分布を持つ架空のRPCレイテンシーを出力する例が含まれている。

[Goコンパイラがインストールされている](https://golang.org/doc/install)ことと[Goのビルド環境](https://golang.org/doc/code.html)が（正しい`GOPATH`で）セットアップされていることを確認する。

PrometheusのGoクライアントライブラリをダウンロードし、3つのサンプルプロセスを実行する

```bash
# Fetch the client library code and compile example.
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go get -d
go build

# Start 3 example targets in separate terminals:
./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

これで、 http://localhost:8080/metrics 、 http://localhost:8081/metrics 、 http://localhost:8082/metrics をリッスンしているサンプルターゲットが出来た。

## <span class="anchor-text-supplement">Configuring </span>サンプルターゲットを監視するためのPrometheusの設定<span class="anchor-text-supplement"> to monitor the sample targets</span>

ここで、新しい監視対象をscrapeするようにPrometheusを設定する。
3つのエンドポイントを`example-random`という1つのジョブにまとめよう。
ただし、最初の2つは本番の監視対象、3つ目はカナリヤだと仮定する。
これをPrometheusで表すために。複数のエンドポイントのグループを1つのジョブに追加し、各グループにラベルを追加することができる。
この例では、ラベル`group="production"`を1つ目のグループに追加し、2つ目には`group="canary"`を追加する。

これを実現するために、以下のジョブの定義を`prometheus.yml`の`scrape_configs`に追加し、Prometheusを再起動する。

```yaml
scrape_configs:
  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

expressionブラウザを開いて、これらのサンプルのエンドポイントが出力している時系列（`rpc_durations_seconds`など）の情報をPrometheusが持っていることを確認する。

## <span class="anchor-text-supplement">Configure rules for aggregating scraped data into new time series</span>スクレイプしたデータを新しい時系列に集約するルールの設定

この例では問題ないが、数千の時系列を集約するクエリは、アドホックに計算すると遅くなることがある。
これをもっと効率的にするために、Prometheusでは、設定されたレコーディングルールを通して、全く新しい永続的な時系列を事前に記録しておくことができる。
例えば、`job`と`service`の軸は残しつつ、5分の窓で全てのインスタンスについての平均にされたサンプルRPCの秒間レート（`rpc_durations_seconds_count`）を記録したいとする。
これは次のように書くことができる。

```
avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

この式をグラフ化してみること。

この式の結果となる時系列を`job_service:rpc_durations_seconds_count:avg_rate5m`というメトリックとして記録するために、以下のレコーディングルールでファイルを作成し、`prometheus.rules.yml`として保存する。

```
groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

このルールをPrometheusが見つけられるように、`rule_files`を`prometheus.yml`に追加する。
これで設定は以下のようになるはずである。

```yaml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  external_labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

新しい設定でPrometheusを再起動して、`job_service:rpc_durations_seconds_count:avg_rate5m`という名前の新しいメトリックが利用可能であることをexpressionブラウザやグラフ化することで確認しよう。