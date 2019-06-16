---
title: ルールの単体テスト
sort_rank: 6
---

# ルールの単体テスト

ルールをテストするために`promtool`を使うことができる。

```shell
# 1つのファイルをテストする
./promtool test rules test.yml

# 複数のファイル、例えばtest1.yml、test2.yml、test2.ymlがある場合
./promtool test rules test1.yml test2.yml test3.yml
```

## テストファイルのフォーマット

```yaml
# テスト対象として扱うルールファイルのリスト
rule_files:
  [ - <file_name> ]

# オプション。デフォルトは1m
evaluation_interval: <duration>

# 以下でグループ名がリストされている順番が（ある時点で）ルールグループが評価される順番となる。
# 以下で記載されているグループのみ順番が保証される。
# 全てのグループが以下に記載されている必要はない。
group_eval_order:
  [ - <group_name> ]

# 全てのテストがここでリストされる
tests:
  [ - <test_group> ]
```

### `<test_group>`

``` yaml
# 時系列データ
interval: <duration>
input_series:
  [ - <series> ]

# 上記データを対象とした単体テスト

# アラートルールに対する単体テスト。入力ファイルからのアラートルールをテストする
alert_rule_test:
  [ - <alert_test_case> ]

# PromQLの式に対する単体テスト
promql_expr_test:
  [ - <promql_test_case> ]
```

### `<series>`

```yaml
# 通常の時系列の記法 <metric name>{<label name>=<label value>, ...} に従う
# 例: 
#      series_name{label1="value1", label2="value2"}
#      go_goroutines{job="prometheus", instance="localhost:9090"}
series: <string>

# 展開される記法を使う
# Expanding notation: 
#     'a+bxc' は 'a a+b a+(2*b) a+(3*b) … a+(c*b)' になる
#     'a-bxc' は 'a a-b a-(2*b) a-(3*b) … a-(c*b)' になる
# 例:
#     1. '-2+4x3' は '-2 2 6 10' になる
#     2. ' 1-2x4' は '1 -1 -3 -5 -7' になる
values: <string>
```

### `<alert_test_case>`

Prometheusでは、異なるアラートルールに同じアラート名をつけることができる。
したがって、この単体テストでは、起きている全てのアラートの和集合を1つの`<alert_test_case>`の中でリストしなければならない。

``` yaml
# アラートがチェックされるべき time=0s からの経過時間
eval_time: <duration>

# テスト対象のアラート名
alertname: <string>

# 与えられた評価時間に与えられたアラート名で起きていると想定しているアラートのリスト。
# アラートが起きているべきではないというテストをしたい場合は、上記フィールドを指定してexp_alertsを空にしておけば良い
exp_alerts:
  [ - <alert> ]
```

### `<alert>`

``` yaml
# これらは想定されるアラートの展開されたラベルとアノテーションである。
# ラベルは、アラートに付随するサンプルのラベルも含む（`/alerts`で見られるものと同じ。ただし`__name__`と`alertname`は除く）。
exp_labels:
  [ <labelname>: <string> ]
exp_annotations:
  [ <labelname>: <string> ]
```

### `<promql_test_case>`

```yaml
# 評価する式
expr: <string>

# time=0s からの経過時間
eval_time: <duration>

# 与えられた評価時間で想定されるサンプル
exp_samples:
  [ - <sample> ]
```

### `<sample>`

```yaml
# 通常の時系列の記法 <metric name>{<label name>=<label value>, ...} でのサンプルのラベル
# 例: 
#      series_name{label1="value1", label2="value2"}
#      go_goroutines{job="prometheus", instance="localhost:9090"}
labels: <string>

# PromQLの式の想定される値
value: <number>
```

## 例

これは、単体テストの入力ファイルのサンプルで、テストにパスする。
`test.yml`は上記の構文に従ったテストファイルで、`alerts.yml`はアラートルールを含んでいる。

`alerts.yml`が同じディレクトリにあるとして、`./promtool test rules test.yml`を実行する。

### `test.yml`

```yaml
# これが、単体テストのメインの入力である。
# このファイルのみがコマンドライン引数として渡される。

rule_files:
    - alerts.yml

evaluation_interval: 1m

tests:
    # Test 1.
    - interval: 1m
      # Series data.
      input_series:
          - series: 'up{job="prometheus", instance="localhost:9090"}'
            values: '0 0 0 0 0 0 0 0 0 0 0 0 0 0 0'
          - series: 'up{job="node_exporter", instance="localhost:9100"}'
            values: '1+0x6 0 0 0 0 0 0 0 0' # 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0
          - series: 'go_goroutines{job="prometheus", instance="localhost:9090"}'
            values: '10+10x2 30+20x5' # 10 20 30 30 50 70 90 110 130
          - series: 'go_goroutines{job="node_exporter", instance="localhost:9100"}'
            values: '10+10x7 10+30x4' # 10 20 30 40 50 60 70 80 10 40 70 100 130

      # アラートルールの単体テスト
      alert_rule_test:
          # Unit test 1.
          - eval_time: 10m
            alertname: InstanceDown
            exp_alerts:
                # Alert 1.
                - exp_labels:
                      severity: page
                      instance: localhost:9090
                      job: prometheus
                  exp_annotations:
                      summary: "Instance localhost:9090 down"
                      description: "localhost:9090 of job prometheus has been down for more than 5 minutes."
      # PromQLの式の単体テスト
      promql_expr_test:
          # Unit test 1.
          - expr: go_goroutines > 5
            eval_time: 4m
            exp_samples:
                # Sample 1.
                - labels: 'go_goroutines{job="prometheus",instance="localhost:9090"}'
                  value: 50
                # Sample 2.
                - labels: 'go_goroutines{job="node_exporter",instance="localhost:9100"}'
                  value: 50
```

### `alerts.yml`

```yaml
# これはルールファイルである

groups:
- name: example
  rules:

  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
        severity: page
    annotations:
        summary: "Instance {{ $labels.instance }} down"
        description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

  - alert: AnotherInstanceDown
    expr: up == 0
    for: 10m
    labels:
        severity: page
    annotations:
        summary: "Instance {{ $labels.instance }} down"
        description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
```
