---
title: 移行
sort_rank: 7
---

# Prometheus 2.0移行ガイド

[安定性の約束](https://prometheus.io/blog/2016/07/18/prometheus-1-0-released/#fine-print)に従って、Prometheus 2.0にはたくさんの後方非互換な変更が含まれている。
このドキュメントは、Prometheus 1.8からPrometheus 2.0に移行するための手引きをする。

## フラグ

Prometheusのコマンドラインフラグの形式が変わった。
1つのハイフンではなく、全てのフラグが2つのハイフンを使う。
よく使われるフラグ（`--config.file`、`--web.listen-address`、`--web.external-url`）はそのままだが、それ以外は、ほぼ全てのストレージ関連のフラグが削除された。

削除されたフラグの中で特筆すべきいくつかを挙げる。

- `-alertmanager.url` Prometheus 2.0では、静的にAlertmanagerのURLを指定するためのコマンドラインフラグは削除された。
  今は、サービスディスカバリーを通してAlertmanagerを検出する必要がある。[Alertmanagerサービスディスカバリー](#amsd)を参照。

- `-log.format` Prometheus 2.0では、標準エラーに流すことしかできない

- `-query.staleness-delta`は`--query.lookback-delta`に名前が変更された。Prometheus 2.0で、データの失効を扱う新しい仕組みが導入された。[失効](querying/basics.md#staleness)を参照。

- `-storage.local.*` Prometheus 2.0で、新しいストレージエンジンが導入され、古いエンジンに関する全てのフラグが削除された。新しいエンジンについての情報は、[ストレージ](#storage)を参照。

- `-storage.remote.*` Prometheus 2.0では、既に非推奨になっていたリモートストレージのフラグが削除された。それらが指定されると起動に失敗するようになる。InfluxDB、Graphite、OpenTSDBに書き込むには、適切なアダプターを使うこと。

## Alertmanagerサービスディスカバリー

Alertmanagerサービスディスカバリーは、Prometheus 1.4で導入され、スクレイプ対象と同じ仕組みでPrometheusがAlertmanagerレプリカを動的に検出できるようになった。
Prometheus 2.0では、静的にAlertmanagerを設定するフラグが削除された。従って、以下のコマンドラインフラグは、

```
./prometheus -alertmanager.url=http://alertmanager:9093/
```

以下の設定ファイル`prometheus.yml`で置き換えられることになるだろう。

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager:9093
```

Alertmanagerの設定では、通常のPrometheusのサービスディスカバリー連携全てとリラベルも利用できる。
以下のスニペットは、Kubernetesの名前空間`default`にあり、ラベル`name: alertmanager`を持ち、ポートが空でないpodをPrometheusに検索するように指示している。

```yaml
alerting:
  alertmanagers:
  - kubernetes_sd_configs:
      - role: pod
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_label_name]
      regex: alertmanager
      action: keep
    - source_labels: [__meta_kubernetes_namespace]
      regex: default
      action: keep
    - source_labels: [__meta_kubernetes_pod_container_port_number]
      regex:
      action: drop
```

## レコーディングルールとアラート

アラートとレコーディングルールを設定する形式がYAMLに変更された。
以下の旧形式のレコーディングルールとアラートの例は、

```
job:request_duration_seconds:histogram_quantile99 =
  histogram_quantile(0.99, sum(rate(request_duration_seconds_bucket[1m])) by (le, job))

ALERT FrontendRequestLatency
  IF job:request_duration_seconds:histogram_quantile99{job="frontend"} > 0.1
  FOR 5m
  ANNOTATIONS {
    summary = "High frontend request latency",
  }
```

次のようになるだろう。

```yaml
groups:
- name: example.rules
  rules:
  - record: job:request_duration_seconds:histogram_quantile99
    expr: histogram_quantile(0.99, sum(rate(request_duration_seconds_bucket[1m]))
      BY (le, job))
  - alert: FrontendRequestLatency
    expr: job:request_duration_seconds:histogram_quantile99{job="frontend"} > 0.1
    for: 5m
    annotations:
      summary: High frontend request latency
```

変更の手助けとなるよう、ツール`promtool`には、ルールの変換を自動化するためのモードがある。
`.rules`ファイルが与えられると、新形式の`.rules.yml`ファイルを出力する。
例えば、以下のように実行する。

```
$ promtool update rules example.rules
```

1.8ではなく2.0のpromtoolを使う必要があることに注意。

## ストレージ

Prometheus 2.0のデータ形式は、根本的に変更され、1.8との後方互換性はない。
過去の監視データを利用し続けるためには、スクレイプをしないPrometheusインスタンスを最低のバージョン１.8.1でPrometheus 2.0と平行して稼働させて、新しいサーバーがremote readプロトコルを通して古いサーバーの既存データを読み込むようにすることを推奨する。

Prometheus 1.8のインスタンスは、以下のフラグと（もしあれば）`external_labels`のみを含む設定ファイルで起動するべきである。

```
$ ./prometheus-1.8.1.linux-amd64/prometheus -web.listen-address ":9094" -config.file old.yml
```

これで、Prometheus 2.0は、（同じマシン上で）以下のフラグで起動できる。

```
$ ./prometheus-2.0.0.linux-amd64/prometheus --config.file prometheus.yml
```

`prometheus.yml`が既存の完全な設定XXX
Where `prometheus.yml` contains in addition to your full existing configuration, the stanza:

```yaml
remote_read:
  - url: "http://localhost:9094/api/v1/read"
```

## PromQL

以下の昨日は、PromQLから削除された。

- `drop_common_labels`関数 - `without`集約修飾子を代わりに使うべきである
- `keep_common`集約修飾子 - `by`修飾子を代わりに使うべきである
- `count_scalar`関数 - `absent()`の方がうまく処理できる。演算でラベルをXXX

- `drop_common_labels` function - the `without` aggregation modifier should be used
  instead.
- `keep_common` aggregation modifier - the `by` modifier should be used instead.
- `count_scalar` function - use cases are better handled by `absent()` or correct
  propagation of labels in operations.

詳細は、[issue #3060](https://github.com/prometheus/prometheus/issues/3060)を参照。

## Miscellaneous

### Prometheus non-root user

PrometheusのDockerイメージは、[root以外のユーザーでPrometheusを実行する](https://github.com/prometheus/prometheus/pull/2859)ようにビルドされている。
PrometheusのUI/APIが小さいポート番号（例えば80番ポート）をリッスンするようにしたい場合、これを上書きする必要がある。
Kubernetesでは、以下のYAMLを使うことになるだろう。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  securityContext:
    runAsUser: 0
...
```

詳細は、[Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)を参照。

Dockerを使っている場合、以下のスニペットが利用されるだろう。

```
docker run -u root -p 80:80 prom/prometheus:v2.0.0-rc.2  --web.listen-address :80
```

### Prometheus lifecycle

[Prometheusの設定が変更された場合に自動的に設定を再読み込みする](configuration/configuration.md)ためにPrometheusのHTTPエンドポイント`/-/reload`を利用するなら、Prometheus 2.0では、セキュリティの理由で、これらのエンドポイントはデフォルトで無効にされている。
これらを有効にするには、フラグ`--web.enable-lifecycle`をセットする。
