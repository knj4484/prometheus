---
title: テンプレートの例
sort_rank: 4
---

# テンプレートの例

Prometheusは、コンソールページと同様に、アラートのアノテーションとラベルでテンプレートをサポートしている。
テンプレートで、ローカルのデータベースに対してクエリを実行したり、繰り返し処理、条件分岐、フォーマットなどをすることが出来る。
Prometheusのテンプレート言語は[Goのテンプレートシステム](https://golang.org/pkg/text/template/)に基づいている。

## 簡単なアラートのフィールドのテンプレート

```
alert: InstanceDown
expr: up == 0
for: 5m
labels:
  severity: page
annotations:
  summary: "Instance {{$labels.instance}} down"
  description: "{{$labels.instance}} of job {{$labels.job}} has been down for more than 5 minutes."
```

アラートのフィールドのテンプレートは、起きたアラートそれぞれに対して各ルールに対する反復処理の中で実行されるので、どのクエリも軽量に保つこと。
アラートのためにより込み入ったテンプレート必要な場合、代わりにコンソールへリンクすることを推奨する。

## 単純な繰り返し

インスタンスのリストとそれがupかどうかのリストを表示する。

```go
{{ range query "up" }}
  {{ .Labels.instance }} {{ .Value }}
{{ end }}
```

特殊変数`.`は、各ループでのサンプル値が含まれる。

## 値の表示

```go
{{ with query "some_metric{instance='someinstance'}" }}
  {{ . | first | value | humanize }}
{{ end }}
```

GoおよびGoのテンプレート言語はどちらも強く型付けされているので、実行エラーを避けるためにサンプルが返されたことをチェックしなければならない。
例えば、スクレイプやルールの評価がまだ実行されていないあるいはホストがダウンしている場合にこうしたことが起きる。

インクルードされたテンプレート`prom_query_drilldown`はこの処理をし、結果をフォーマットし、[expressionブラウザ](https://prometheus.io/docs/visualization/browser/)にリンクする。

## コンソールURLパラメーターの利用

```go
{{ with printf "node_memory_MemTotal{job='node',instance='%s'}" .Params.instance | query }}
  {{ . | first | value | humanize1024 }}B
{{ end }}
```

`console.html?instance=hostname`としてアクセスされると、`.Params.instance`は、`hostname`になる。

## 高度な反復処理

```html
<table>
{{ range printf "node_network_receive_bytes{job='node',instance='%s',device!='lo'}" .Params.instance | query | sortByLabel "device"}}
  <tr><th colspan=2>{{ .Labels.device }}</th></tr>
  <tr>
    <td>Received</td>
    <td>{{ with printf "rate(node_network_receive_bytes{job='node',instance='%s',device='%s'}[5m])" .Labels.instance .Labels.device | query }}{{ . | first | value | humanize }}B/s{{end}}</td>
  </tr>
  <tr>
    <td>Transmitted</td>
    <td>{{ with printf "rate(node_network_transmit_bytes{job='node',instance='%s',device='%s'}[5m])" .Labels.instance .Labels.device | query }}{{ . | first | value | humanize }}B/s{{end}}</td>
  </tr>{{ end }}
</table>
```

ここでは、全ネットワークデバイスについて反復処理をして、それぞれのネットワークトラフィックを表示している。

`range`アクションが変数を指定していないので、`.`がループ変数となり、ループ内では`.Params.instance`は利用できない。

## 再利用可能なテンプレートの定義

Prometheusは、再利用可能なテンプレートの定義をサポートしている。
これは[コンソールライブラリ](template_reference.md#console-templates)と組み合わせると特に強力で、複数のコンソールにまたがってテンプレートを共有出来るようになる。

```go
{{/* Define the template */}}
{{define "myTemplate"}}
  do something
{{end}}

{{/* Use the template */}}
{{template "myTemplate"}}
```

テンプレートは、引数が1つに制限されている。複数の引数をまとめるために、関数`args`を利用することが出来る。

```go
{{define "myMultiArgTemplate"}}
  First argument: {{.arg0}}
  Second argument: {{.arg1}}
{{end}}
{{template "myMultiArgTemplate" (args 1 2)}}
```
