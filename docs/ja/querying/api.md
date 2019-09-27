---
title: HTTP API
sort_rank: 7
---

# HTTP API

現状の安定版HTTP APIは、Prometheusサーバーの`/api/v1`でアクセスできる。 非破壊的な追加がこのエンドポイントに追加されることがある。

## フォーマット概要

成功したAPIリクエストはステータスコード`2xx`を返す。

APIハンドラに届いた不正なリクエストは、JSONエラーオブジェクトと以下のHTTPレスポンスコードの1つを返す。

- `400 Bad Request` パラメーターが欠けていたり間違っている場合
- `422 Unprocessable Entity` 式が実行できない場合([RFC4918](https://tools.ietf.org/html/rfc4918#page-78))
- `503 Service Unavailable` クエリがタイムアウトしたりアボートした場合

APIエンドポイントに到達する前に起きたエラーに対しては、`2xx`でない他のコードが返される場合もある。

リクエストの実行を妨げなかったエラーがあると、警告の配列が返される場合もある。
収集に成功した全てのデータは、データフィールド入れて返される。

JSONレスポンスのフォーマットは以下の通りである。

```
{
  "status": "success" | "error",
  "data": <data>,

  // statusがerrorの場合のみセットされる。
  // その場合でも、dataフィールドは追加のデータを保持している場合がある。
  "errorType": "<string>",
  "error": "<string>",

  // リクエスト実行中に警告があった場合のみセットされる。
  // その場合でも、dataフィールドは追加のデータを保持している場合がある。
  "warnings": ["<string>"]
}
```

入力のタイムスタンプは、[RFC3339](https://www.ietf.org/rfc/rfc3339.txt)のフォーマットまたは秒で表したUnixタイムスタンプ（オプションで秒未満の精度のために少数をつける）として与えることができる。 出力のタイムスタンプは、常に、秒で表されたUnixタイムスタンプである。

繰り返される可能性があるクエリパラメーターの名前は[]で終わる。

プレースホルダー`<series_selector>`は、`http_requests_total`や`http_requests_total{method=~"(GET|POST)"}`のような時系列セレクターを表し、URLエンコードされてなければいけない。

プレースホルダー`<duration>`は、`[0-9]+[smhdwy]`という形式の時間幅の文字列を表す。 例えば、`5m`は5分の時間幅を表す。

プレースホルダー`<bool>`は、ブール値（文字列で`true`と`false`）を表す。

## 式のクエリ

クエリ言語の式は、時間のある一点で評価されるかもしれないし、時間幅に渡って評価されるかもしれない。
以下のセクションでは、それぞれのタイプのクエリのためのAPIエンドポイントについて説明する。

### Instant queries

以下のエンドポイントは、時間の一点でのinstantクエリを評価する。

```
GET /api/v1/query
POST /api/v1/query
```

URLクエリパラメーターは以下の通り。

- `query=<string>`: Prometheusのexpressionのクエリの文字列
- `time=<rfc3339 | unix_timestamp>`: 評価時間（オプション）
- `timeout=<duration>`: 評価のタイムアウト（オプション）。`-query.timeout`フラグの値がデフォルトであり、かつ上限となる。

`time`パラメーターが省略されると、現在のサーバー時間が利用される。

`POST`メソッドと`Content-Type: application/x-www-form-urlencoded`ヘッダを利用することで、リクエストボディの中でこれらのパラメーターをURLエンコードすることができる。
これは、サーバーサイドの文字制限に反する巨大なクエリを指定する際に便利である。

結果の`data`セクションは以下のフォーマットである。

```
{
  "resultType": "matrix" | "vector" | "scalar" | "string",
  "result": <value>
}
```

`<value>`は、クエリの結果のデータを表しており、`resultType`に応じて、様々なフォーマットになる。 後述の[結果のフォーマット](#expression-query-result-formats)を参照すること。

以下の例は、式upを`2015-07-01T20:10:51.781Z`の時点で評価している。

```json
$ curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
{
   "status" : "success",
   "data" : {
      "resultType" : "vector",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "value": [ 1435781451.781, "1" ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9100"
            },
            "value" : [ 1435781451.781, "0" ]
         }
      ]
   }
}
```

### Range queries

以下のエンドポイントは、ある時間幅に渡ってクエリの式を評価する。

```
GET /api/v1/query_range
POST /api/v1/query_range
```

URLクエリパラメーターは以下の通り。

- `query=<string>`: Prometheusのexpressionのクエリの文字列
- `start=<rfc3339 | unix_timestamp>`: 開始時刻のタイムスタンプ
- `end=<rfc3339 | unix_timestamp>`: 終了時刻のタイムスタンプ
- `step=<duration | float>`: クエリ解像度のステップ幅（`duration`フォーマットまたは秒数の少数）
- `timeout=<duration>`: 評価のタイムアウト（オプション）。`-query.timeout`フラグの値がデフォルトであり、かつ上限となる。

`POST`メソッドと`Content-Type: application/x-www-form-urlencoded`ヘッダを利用することで、リクエストボディの中でこれらのパラメーターをURLエンコードすることができる。
これは、サーバーサイドの文字制限に反する巨大なクエリを指定する際に便利である。

結果の`data`セクションは以下のフォーマットである。

```
{
  "resultType": "matrix",
  "result": <value>
}
```

プレイスホルダー`<value>`のフォーマットについては、[range-vector結果フォーマット](#range-vectors)を参照すること。

以下の例は、式upを30秒の時間幅にかけて15秒の解像度で評価している。

```json
$ curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
{
   "status" : "success",
   "data" : {
      "resultType" : "matrix",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "values" : [
               [ 1435781430.781, "1" ],
               [ 1435781445.781, "1" ],
               [ 1435781460.781, "1" ]
            ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9091"
            },
            "values" : [
               [ 1435781430.781, "0" ],
               [ 1435781445.781, "0" ],
               [ 1435781460.781, "1" ]
            ]
         }
      ]
   }
}
```

## メタデータのクエリ

### ラベルマッチャーによる時系列の検索

以下のエンドポイントはラベル集合にマッチする時系列のリストを返す。

```
GET /api/v1/series
POST /api/v1/series
```

URLクエリパラメーターは以下の通り。

- `atch[]=<series_selector>`: 時系列を選択するためのセレクター引数。少なくとも1つのmatch[]を指定する必要がある
- `start=<rfc3339 | unix_timestamp>`: 開始時刻のタイムスタンプ
- `end=<rfc3339 | unix_timestamp>`: 終了時刻のタイムスタンプ

`POST`メソッドと`Content-Type: application/x-www-form-urlencoded`ヘッダを利用することで、リクエストボディの中でこれらのパラメーターをURLエンコードすることができる。
これは、サーバーサイドの文字制限に反する巨大なクエリを指定する際に便利である。

結果のdataセクションは、各時系列を特定するラベル名/値を含むオブジェクトのリストである。

以下の例は、セレクター`up`または`process_start_time_seconds{job="prometheus"}`にマッチする全ての時系列を返す。

```json
$ curl -g 'http://localhost:9090/api/v1/series?' --data-urlencode='match[]=up' --data-urlencode='match[]=process_start_time_seconds{job="prometheus"}'
{
   "status" : "success",
   "data" : [
      {
         "__name__" : "up",
         "job" : "prometheus",
         "instance" : "localhost:9090"
      },
      {
         "__name__" : "up",
         "job" : "node",
         "instance" : "localhost:9091"
      },
      {
         "__name__" : "process_start_time_seconds",
         "job" : "prometheus",
         "instance" : "localhost:9090"
      }
   ]
}
```

### ラベル名の取得

以下のエンドポイントはラベル名のリストを返す。

```
GET /api/v1/labels
POST /api/v1/labels
```

JSONレスポンスの`data`セクションは、ラベル名の文字列のリストである。

以下に例を示す。

```json
$ curl 'localhost:9090/api/v1/labels'
{
    "status": "success",
    "data": [
        "__name__",
        "call",
        "code",
        "config",
        "dialer_name",
        "endpoint",
        "event",
        "goversion",
        "handler",
        "instance",
        "interval",
        "job",
        "le",
        "listener_name",
        "name",
        "quantile",
        "reason",
        "role",
        "scrape_job",
        "slice",
        "version"
    ]
}
```

### ラベル値のクエリ

以下のエンドポイントは、与えられたラベル名に対するラベル値のリストを返す。

```
GET /api/v1/label/<label_name>/values
```

JSONレスポンスの`data`セクションは、ラベル値の文字列のリストである。

この例は、ラベル`job`に対する全てのラベル値を取得している。

```json
$ curl http://localhost:9090/api/v1/label/job/values
{
   "status" : "success",
   "data" : [
      "node",
      "prometheus"
   ]
}
```

## 式のクエリの結果フォーマット

式のクエリは、`data`セクションのresultプロパティに以下のレスポンスを入れて返す。 プレースホルダー`<sample_value>`は、数値のサンプル値である。 `JSON`は、`NaN`や`Inf`、`-Inf`といった特別な値をサポートしないので、生の数値の代わりに、クオートされたJSON文字列として送信される。

### Range vectors

Range vectorは、結果タイプ`matrix`として返される。 対応する`result`プロパティは、以下のフォーマットとなる。

```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "values": [ [ <unix_time>, "<sample_value>" ], ... ]
  },
  ...
]
```

### Instant vectors

Instant vectorは、結果タイプ`vector`として返される。 対応する`result`プロパティは、以下のフォーマットとなる。

```
[
  {
    "metric": { "<label_name>": "<label_value>", ... },
    "value": [ <unix_time>, "<sample_value>" ]
  },
  ...
]
```

### スカラー

スカラーは、結果タイプ`scalar`として返される。 対応する`result`プロパティは、以下のフォーマットとなる。

```
[ <unix_time>, "<scalar_value>" ]
```

### 文字列

文字列は、結果タイプ`string`として返される。 対応する`result`プロパティは、以下のフォーマットとなる。

```
[ <unix_time>, "<string_value>" ]
```

## 監視対象

以下のエンドポイントは、監視対象の検出の現在の状態の概要を返す。

```
GET /api/v1/targets
```

activeなターゲットとdropされたターゲットのどちらも含まれる。 `labels`は、リラベルが起きた後のラベル集合を表す。
`discoveredLabels`は、サービスディスカバリーで取得したリラベリングされる前の修正されていないラベルを表す。

```json
$ curl http://localhost:9090/api/v1/targets
{
  "status": "success",
  "data": {
    "activeTargets": [
      {
        "discoveredLabels": {
          "__address__": "127.0.0.1:9090",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "job": "prometheus"
        },
        "labels": {
          "instance": "127.0.0.1:9090",
          "job": "prometheus"
        },
        "scrapeUrl": "http://127.0.0.1:9090/metrics",
        "lastError": "",
        "lastScrape": "2017-01-17T15:07:44.723715405+01:00",
        "health": "up"
      }
    ],
    "droppedTargets": [
      {
        "discoveredLabels": {
          "__address__": "127.0.0.1:9100",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "job": "node"
        },
      }
    ]
  }
}
```

## ルール

APIエンドポイント`/rules`は、現在読み込まれているアラートルールとレコーディングルールのリストを返す。 Prometheusが起こした各アラートルールの現在activeなアラートも返す。

エンドポイント`/rules`は、かなり新しいので、API v1と同じ安定性の保証はない。

```
GET /api/v1/rules
```

```json
$ curl http://localhost:9090/api/v1/rules

{
    "data": {
        "groups": [
            {
                "rules": [
                    {
                        "alerts": [
                            {
                                "activeAt": "2018-07-04T20:27:12.60602144+02:00",
                                "annotations": {
                                    "summary": "High request latency"
                                },
                                "labels": {
                                    "alertname": "HighRequestLatency",
                                    "severity": "page"
                                },
                                "state": "firing",
                                "value": "1e+00"
                            }
                        ],
                        "annotations": {
                            "summary": "High request latency"
                        },
                        "duration": 600,
                        "health": "ok",
                        "labels": {
                            "severity": "page"
                        },
                        "name": "HighRequestLatency",
                        "query": "job:request_latency_seconds:mean5m{job=\"myjob\"} > 0.5",
                        "type": "alerting"
                    },
                    {
                        "health": "ok",
                        "name": "job:http_inprogress_requests:sum",
                        "query": "sum(http_inprogress_requests) by (job)",
                        "type": "recording"
                    }
                ],
                "file": "/rules.yaml",
                "interval": 60,
                "name": "example"
            }
        ]
    },
    "status": "success"
}
```


## アラート

エンドポイント`/alerts`は、全てのactiveなアラートのリストを返す。

エンドポイント`/alerts`はかなり新しいので、API v1と同じ安定性の保証はない。

```
GET /api/v1/alerts
```

```json
$ curl http://localhost:9090/api/v1/alerts

{
    "data": {
        "alerts": [
            {
                "activeAt": "2018-07-04T20:27:12.60602144+02:00",
                "annotations": {},
                "labels": {
                    "alertname": "my-alert"
                },
                "state": "firing",
                "value": "1e+00"
            }
        ]
    },
    "status": "success"
}
```

## Querying target metadata

下記のエンドポイントは、現在取得ずみのメトリクスに関するメタデータを返す。 これは、**実験的**であり、将来的には変更される可能性がある。

```
GET /api/v1/targets/metadata
```

URLクエリパラメーターは以下の通り。

- `match_target=<label_selectors>`: ラベル集合でターゲットとマッチングをするラベルセレクター。空にすると全てのターゲットが選択される
- `metric=<string>`: メタデータを取得する対象のメトリック名。空にすると全てのメトリックのメタデータが選択される
- `limit=<number>`: マッチングするターゲットの最大数

クエリの結果の`data`部分は、メトリックのメタデータとターゲットのラベル集合を含むオブジェクトのリストから成る。

以下の例は、最初の2つのターゲットのメトリック`go_goroutines`でラベル`job="prometheus"`を持つものに対する全てのメタデータを返す。

```json
curl -G http://localhost:9091/api/v1/targets/metadata \
    --data-urlencode 'metric=go_goroutines' \
    --data-urlencode 'match_target={job="prometheus"}' \
    --data-urlencode 'limit=2'
{
  "status": "success",
  "data": [
    {
      "target": {
        "instance": "127.0.0.1:9090",
        "job": "prometheus"
      },
      "type": "gauge",
      "help": "Number of goroutines that currently exist.",
      "unit": ""
    },
    {
      "target": {
        "instance": "127.0.0.1:9091",
        "job": "prometheus"
      },
      "type": "gauge",
      "help": "Number of goroutines that currently exist.",
      "unit": ""
    }
  ]
}
```

以下の例は、ラベル`instance="127.0.0.1:9090"`を持つ全てのターゲットの全てのメトリックに対するメタデータを返す。

```json
curl -G http://localhost:9091/api/v1/targets/metadata \
    --data-urlencode 'match_target={instance="127.0.0.1:9090"}'
{
  "status": "success",
  "data": [
    // ...
    {
      "target": {
        "instance": "127.0.0.1:9090",
        "job": "prometheus"
      },
      "metric": "prometheus_treecache_zookeeper_failures_total",
      "type": "counter",
      "help": "The total number of ZooKeeper failures.",
      "unit": ""
    },
    {
      "target": {
        "instance": "127.0.0.1:9090",
        "job": "prometheus"
      },
      "metric": "prometheus_tsdb_reloads_total",
      "type": "counter",
      "help": "Number of times the database reloaded block data from disk.",
      "unit": ""
    },
    // ...
  ]
}
```

## Alertmanagers

以下のエンドポイントは、Alertmanager検出の現在の状況の概要を返す。

```
GET /api/v1/alertmanagers
```

activeなAlertmanagerとdropされたAlertmanagerのどちらも結果に含まれる。

```json
$ curl http://localhost:9090/api/v1/alertmanagers
{
  "status"593
  "success",
  "data": {
    "activeAlertmanagers": [
      {
        "url": "http://127.0.0.1:9090/api/v1/alerts"
      }
    ],
    "droppedAlertmanagers": [
      {
        "url": "http://127.0.0.1:9093/api/v1/alerts"
      }
    ]
  }
}
```

## Status

以下のstatusエンドポイントは、Prometheusの設定を出力する。

### Config

以下のエンドポイントは、現在読み込まれている設定ファイルを返す。

```
GET /api/v1/status/config
```

設定は、YAMLファイルのダンプとして返される。YAMLライブラリの制限で、YAMLコメントは含まれない。

```json
$ curl http://localhost:9090/api/v1/status/config
{
  "status": "success",
  "data": {
    "yaml": "<content of the loaded config file in YAML>",
  }
}
```

### Flags

以下のエンドポイントは、Prometheusに与えられたフラグの値を返す。

```
GET /api/v1/status/flags
```

全ての値は文字列である。

```json
$ curl http://localhost:9090/api/v1/status/flags
{
  "status": "success",
  "data": {
    "alertmanager.notification-queue-capacity": "10000",
    "alertmanager.timeout": "10s",
    "log.level": "info",
    "query.lookback-delta": "5m",
    "query.max-concurrency": "20",
    ...
  }
}
```

*New in v2.2*

## TSDB Admin APIs
これらは、上級ユーザーのためにデータベースの機能を出力するAPIである。 `--web.enable-admin-api`がセットされていなければ有効ではない。

gRPC APIも公開しており、[ここ](https://github.com/prometheus/prometheus/blob/master/prompb/rpc.proto)に定義がある。 これは、実験的なものであり、将来的には変更される可能性がある。

### スナップショット
スナップショットは、TSDBのデータディレクトリの下の`snapshots/<datetime>-<rand>`に全ての現在のデータのスナップショットを作成し、そのディレクトリを返す。 
オプションで、ヘッドブロックだけにありまだディスクにコンパクト化されていないデータのスナップショットをスキップする。

```
POST /api/v1/admin/tsdb/snapshot
PUT /api/v1/admin/tsdb/snapshot
```

URLクエリパラメーターは以下の通り。

- `skip_head=<bool>`: headブロックにあるデータをスキップする。オプション

```json
$ curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot
{
  "status": "success",
  "data": {
    "name": "20171210T211224Z-2be650b6d019eb54"
  }
}
```

このスナップショットは`<data-dir>/snapshots/20171210T211224Z-2be650b6d019eb54`にある。

*New in v2.1 and supports PUT from v2.9*

### 時系列の削除
DeleteSeriesは、ある時間幅にある選択された時系列のデータを削除する。 実際のデータはディスクに存在し続けて、その後のコンパクションで片付けられるか、Clean Tombstonesエンドポイントを叩くことで片付けられる。

成功すると`204`が返される。

```
POST /api/v1/admin/tsdb/delete_series
PUT /api/v1/admin/tsdb/delete_series
```

URLクエリパラメーターは以下の通り。

- `match[]=<series_selector>`: 時系列を選択するためのセレクター引数。少なくとも1つのmatch[]を指定する必要がある
- `start=<rfc3339 | unix_timestamp>`: 開始時刻のタイムスタンプ。オプションでデフォルトは可能な限り最小の時刻
- `end=<rfc3339 | unix_timestamp>`: 終了時刻のタイムスタンプ。オプションでデフォルトは可能な限り最大の時刻

開始時刻と終了時刻を指定しないと、データベース中のマッチした時系列の全てのデータが削除される。

例

```json
$ curl -X POST \
  -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=up&match[]=process_start_time_seconds{job="prometheus"}'
```
*New in v2.1 and supports PUT from v2.9*

### tombstoneの削除
CleanTombstonesは、削除済みのデータをディスクから削除し、存在するtombstonesを一掃する。 これは、時系列を削除した後に、ディスクスペースを空けるために利用される。

成功すると`204`が返される。

```
POST /api/v1/admin/tsdb/clean_tombstones
PUT /api/v1/admin/tsdb/clean_tombstones
```

パラメーターやレスポンスボディはない。

```json
$ curl -XPOST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

*New in v2.1 and supports PUT from v2.9*
