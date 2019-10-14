---
title: アラートルール
sort_rank: 3
---

# <span class="anchor-text-supplement">Alerting rules</span>アラートルール

アラートルールによって、Prometheusの式に基づいたアラート条件を定義し、外部サービスにアラートに関する通知を送信できるようになる。
ある時点での式の結果が1つ以上のベクトル要素になれば、それらの要素のラベル集合に対してアラートがactiveであるとみなされる。

### <span class="anchor-text-supplement">Defining alerting rules</span>アラートルールの定義

アラートルールは、[レコーディングルール](recording_rules.md)と同じ方法で設定される。

1つのアラートを含むルールの例は、以下のようになるだろう。

```yaml
groups:
- name: example
  rules:
  - alert: HighRequestLatency
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
```

`for`によって、初めて出力ベクトルの要素があった時から一定時間Prometheusを待機させ、アラートをfiringとみなす。
この例では、Prometheusは、アラートが10分間の各評価でずっとアクティブであることを確認してからアラートをfiringにする。
activeではあるがfiringでない要素は、pending状態である。

`labels`によって、アラートに追加的なラベル集合を付与できる。
既にあるラベルと衝突した場合、既存ラベルが上書きされる。
ラベルの値には、テンプレートが利用できる。

`annotations`は、アラート詳細や手順書へのリンクなどの長めの情報を追加するために利用されるラベルを指定する。
アノテーションの値には、テンプレートが利用できる。

#### <span class="anchor-text-supplement">Templating</span>テンプレート

ラベルとアノテーションの値には[コンソールテンプレート](https://prometheus.io/docs/visualization/consoles)が利用できる。
`$labels`は、アラートのラベルのキーバリューを保持し、`$value`は、アラートの評価値を保持している。

ラベルとアノテーションの値には[コンソールテンプレート](https://prometheus.io/docs/visualization/consoles)が利用できる。
`$labels`は、アラートのラベルのキーバリューを保持している。
設定された外部ラベルは`$externalLabels`を通して利用可能である。
`$value`は、アラートの評価値を保持している。

    # firingな要素のラベル値を挿入するには
    {{ $labels.<labelname> }}
    # firingな要素の数値表現を挿入するには
    {{ $value }}

例

```yaml
groups:
- name: example
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

  # Alert for any instance that has a median request latency >1s.
  - alert: APIHighRequestLatency
    expr: api_http_request_latencies_second{quantile="0.5"} > 1
    for: 10m
    annotations:
      summary: "High request latency on {{ $labels.instance }}"
      description: "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)"
```

### <span class="anchor-text-supplement">Inspecting alerts during runtime</span>実行時のアラートの調査

どのアラートがactive（pendingまたはfiring）であるかを手動で調べるためには、PrometheusのAlertsタブにアクセスする。
ここには、定義されたアラートそれぞれのがどのようなラベル集合に対して現在activeかが表示される。

pendingとfiringのアラートに関して、Prometheusは、`ALERTS{alertname="<alert name>", alertstate="pending|firing", <additional alert labels>}`という形式の時系列も保存する。
アラートが指定されたactiveな状態（pendingまたはfiring）の間は値が1にセットされ、そうでなくなればstaleとみなされる。

### <span class="anchor-text-supplement">Sending alert notifications</span>アラート通知の送信

Prometheusのアラートルールは、**今**何が壊れているかを把握するのは得意であるが、本格的な通知のソリューションではない。
ここで説明した簡単なアラートの定義を元にして、アラートをまとめたり、通知レートを制限したり、抑制したり、アラート同士の依存関係を定義したりするためには、別のレイヤーが必要である。
Prometheusエコシステムでは、[Alertmanager](https://prometheus.io/docs/alerting/alertmanager/)がこの役割を担う。
したがって、Prometheusは、定期的にアラートの状態に関する情報をAlertmanagerに送信するように設定することができる。
Alertmanagerは、正しい通知の送信処理をする。
Prometheusは、サービスディスカバリー連携を通して利用可能なAlertmanagerを自動的に検出するように[設定](configuration.md)することができる。
