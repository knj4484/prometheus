---
title: レコーディングルール
sort_rank: 2
---

# <span class="anchor-text-supplement">Defining recording rules</span>レコーディングルール定義

## <span class="anchor-text-supplement">Configuring rules</span>ルール設定

Prometheusは、2種類のルール（レコーディングルールと[アラートルール](alerting_rules.md)）をサポートしており、それらは一定の時間間隔で評価される。
Prometheusにルールを取り込むには、必要なルールを含んだファイルを作成し、[Prometheusの設定項目](configuration.md)`rule_files`を通してファイルを読み込ませる。
ルールのファイルにはYAMLを利用する。

ルールファイルは、Prometheusプロセスに`SIGHUP`を送ることで、実行時に再読み込みが出来る。
全てのルールファイルが正しくフォーマットされている場合にのみ、変更が適用される。

## <span class="anchor-text-supplement">Syntax-checking rules</span>ルールの構文チェック

ルールファイルが構文的に正しいかを、Prometheusサーバーを起動せずにさっと確認するには、Prometheusのコマンドラインユーティリティ`promtool`をインストール、実行する。

```bash
go get github.com/prometheus/prometheus/cmd/promtool
promtool check rules /path/to/example.rules.yml
```

ファイルが構文的に正しい時には、チェッカーは、パースしたルールをテキストとして標準出力に出力し、リターンコード`0`で終了する。

何かしら構文エラーがあるか、入力引数が不正な場合は、標準エラーにエラーメッセージを出力し、リターンコード`1`で終了する。

## <span class="anchor-text-supplement">Recording rules</span>レコーディングルール

レコーディングルールによって、頻繁に必要であったり計算量の多い式を事前に計算し、新しい時系列として結果を保存することが出来る。
事前に計算された結果に対するクエリは、元の式を必要な時に毎回実行するよりも格段に早いことがよくある。
これは、リフレッシュするたびに繰り返し同じ式をクエリする必要があるダッシュボードのために特に有益である。

recording ruleとalerting ruleは、一つのルールグループに紐づけられる。
あるグループ内の複数のルールは、一定の時間間隔で逐次的に実行される。

ルールファイルの構文は以下の通り。

```yaml
groups:
  [ - <rule_group> ]
```

ルールファイルの簡単な例は次のようになるだろう。

```yaml
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

### `<rule_group>`
```
# グループの名前。ファイルの中でユニークでなければならない。
name: <string>

# グループ内のルールがどれぐらい頻繁に評価されるか
[ interval: <duration> | default = global.evaluation_interval ]

rules:
  [ - <rule> ... ]
```

### `<rule>`

レコーディングルールの構文は以下の通り。

```
# 出力する時系列の名前。正当なメトリック名でなければならない
record: <string>

# 評価するPromQLの式。評価の各サイクルの時点でこの式が評価され、recordで指定されたメトリック名で
# 新しい時系列の集合として結果が記録される。
expr: <string>

# 結果を記録する前に追加または上書きされるラベル
labels:
  [ <labelname>: <labelvalue> ]
```

アラートルールの構文は以下の通り。

```
# アラート名。正当なメトリック名でなければならない
alert: <string>

# 評価するPromQLの式。評価の各サイクルの時点でこの式が評価され、全ての結果の時系列が
# pending/firingされたアラートになる
expr: <string>

# この長さに渡ってアラートが返されると、firingとみなされる。
# この長さまで起き続けていないアラートは、pendingとみなされる。
[ for: <duration> | default = 0s ]

# 各アラートに対して追加または上書きされるラベル
labels:
  [ <labelname>: <tmpl_string> ]

# 各アラートに追加されるアノテーション
annotations:
  [ <labelname>: <tmpl_string> ]
```
