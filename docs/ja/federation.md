---
title: フェデレーション
sort_rank: 6
---

# フェデレーション

Prometheusサーバーは、フェデレーションによって、他のPrometheusサーバーから時系列データを選択的に取得することが出来る。

## ユースケース

フェデレーションには様々なユースケースがある。
スケーラブルなPrometheusの監視の配置を達成するため、または、あるサービスのPrometheusから他のPrometheusに関連のあるメトリクスをpullするために使うのが一般的である。

### 階層的なフェデレーション

階層的なフェデレーションによって、Prometheusは、何十のデータセンターと何百万のノードから成る環境にスケールすることが出来る。
このユースケースでは、フェデレーションの繋ぎ方は木構造のようになる。高レベルのPrometheusサーバーほど、下位のより多くのサーバーから集約された時系列データを収集する。

例えば、ある配置は、インスタンスレベルの詳細なデータを集めるデータセンターごとのたくさんのPrometheusサーバーと、ジョブレベルの集約されたデータのみをそれらのPrometheusから収集・保存するグローバルなPrometheusサーバーから構成されるだろう。これによって、集約された大局的な見方も、局所的な詳細な見方も出来るようになる。

### サービス間のフェデレーション

サービス間のフェデレーションでは、あるサービスのPrometheusサーバーが、他のサービスのPrometheusサーバーから選択したデータを収集するように設定され、一つのサービスの中で両方のデータに対するアラートとクエリが可能になる。

例えば、複数のサービスを実行中のクラスタスケジューラーは、そのクラスタ内のサービスインスタンスのリソース使用量（メモリやCPU利用量）の情報をexposeしているだろう。
他方で、そのクラスタで実行中のサービスはアプリケーション特有のサービスのメトリクスのみをexposeしているだろう。
多くの場合、これらの二つのメトリクスは別々のPrometheusサーバーが取得している。
フェデレーションを使うと、サービスのメトリクスを持っているPrometheusサーバーは、その特定のサービスに関するクラスタリソース使用量をクラスタのPrometheusサーバーから取得し、両方のメトリクスがそのサーバーで利用できるようになる。

## フェデレーションの設定

どのPrometheusサーバでも、エンドポイント`/federate`によってそのサーバーのその時点の時系列の値を取得できる。
時系列データを選択するためにURLパラメーターで少なくとも一つの`match[]`が指定されていなければならない。
それぞれの`match[]`の引数は、`up`や`{job="api-server"}`のような[instance vector selector](querying/basics.md#instant-vector-selectors)を指定しなければいけない。

あるサーバーから他のサーバーにメトリクスをfederateするためには、受け取り側のPrometheusサーバーが提供側のPrometheusサーバーのエンドポイント`/federate`をスクレイプするようにすると同時に、取得したラベルを上書きしないように`honor_labels`を有効にし、必要な`match[]`パラメーターを渡すように設定すること。
例えば、以下の`scrape_configs`は、ラベルが`job="prometheus"`またはメトリック名が`job:`で始まる時系列データを`source-prometheus-{1,2,3}:9090`から取得する。

```yaml
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s

    honor_labels: true
    metrics_path: '/federate'

    params:
      'match[]':
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'

    static_configs:
      - targets:
        - 'source-prometheus-1:9090'
        - 'source-prometheus-2:9090'
        - 'source-prometheus-3:9090'
```
