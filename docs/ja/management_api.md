---
title: 管理用API
sort_rank: 7
---

# <span class="original-header">Management API</span>

Prometheusは、自動化および連携を容易にするために管理用API一式を提供する。


### <span class="original-header">Health check</span>ヘルスチェック

```
GET /-/healthy
```

このエンドポイントは常に200を返し、Prometheusの状態を確認するために使われるべきである。


### <span class="original-header">Readiness check</span>Readiness check

```
GET /-/ready
```

このエンドポイントはPrometheusがトラフィックを処理できる（クエリに答える）準備が出来ている時に200を返す。


### <span class="original-header">Reload</span>再読み込み

```
PUT  /-/reload
POST /-/reload
```

このエンドポイントは、Prometheusの設定とルールファイルの再読み込みを引き起こす。デフォルトでは無効になっており、`--web.enable-lifecycle`フラグで有効にすることが出来る。

設定の再読み込みを引き起こす別の方法は、`SIGHUP`をPrometheusプロセスに送ることである。


### <span class="original-header">Quit</span>終了

```
PUT  /-/quit
POST /-/quit
```

このエンドポイントは、Prometheusの正常なシャットダウン（graceful shutdown）を引き起こす。デフォルトでは無効になっており、`--web.enable-lifecycle`フラグで有効にすることが出来る。

正常なシャットダウンを引き起こす別の方法は、`SIGTERM`をPrometheusプロセスに送ることである。
