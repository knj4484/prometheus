# 内部構造

Prometheusサーバーは、Prometheus全体の機能を生み出すために協調して一緒に働く多くの内部コンポーネントから成る。
このドキュメントは、それらがどのように組み合わさっているかを含む主要なコンポーネントの概要およびその実装の詳細へのリンクを示す。
もしPrometheusの新米開発者でPrometheusの各部分の概要を知りたいのなら、ここからはじめよう。

このダイアグラムで、Prometheusサーバー全体の構成が示されている。

![Prometheus server architecture](images/internal_architecture.svg)

**注**: 矢印は、リクエストやコネクションを開始する方向を示し、必ずしもデータの流れの方向ではない。

以下のセクションで、図中の各コンポーネントについて説明する。
コードへのリンクや説明は、Prometheus 2.3.1に基づいている。
Prometheusの将来のバージョンでは異なる可能性がある。

## main関数

[`main()` function](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L77-L600)は、Prometheusサーバーの他の全てのコンポーネントを初期化し稼働させる。
また、依存関係のあるコンポーネント同士を繋げる。

第一歩として、`main()`は、サーバーのコマンドラインフラグを定義し、解析して[ローカルの設定の構造体](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L83-L100)に変換する。
同時に、サニタイズやいくつかの値の初期化を行う。
フラグに基づくこの設定の構造体は、あとで（`--config.file`フラグで指定された）設定ファイルから読み込まれる設定からは独立していることに注意すること。
Prometheusは、フラグに基づく設定とファイルに基づく設定を区別する。フラグは、サーバーの再起動なしに更新することができない単純な設定利用され流のに対し、設定ファイルで提供されている設定項目は、サーバーを再起動せずに再読み込みできなければならない。

次に、`main()`は、Prometheusの実行時のコンポーネントを初期化し、channelや参照、後の協調動作やキャンセル処理のためのcontextの受け渡しを利用してそれらを接続する。
これらのコンポーネントには、（このドキュメントの残りで説明されているように）サービスディスカバリー、監視対象のスクレイプ、ストレージなどが含まれる。

最後にPrometheusサーバーは、起動とシャットダウンを調整するために[`github.com/oklog/oklog/pkg/group`](https://godoc.org/github.com/oklog/run)を利用して、[actorモデル](https://www.brianstorti.com/the-actor-model/)のように[全てのコンポーネントを実行](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L366-L598)する。
順序の制約を為に強制する（ストレージの準備が出来て最初の設定ファイル読み込みが起きるまではwebインターフェースを有効にしないなど）ために、複数のチャネルが利用される。

## 設定

設定のサブシステムは、`--config.file`フラグで指定されたYAMLの設定ファイルで与えられる設定の読み込み、検証、適用を受け持つ。
全ての設定項目の説明は[ドキュメント](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)を参照すること。
Prometheusには、設定ファイルの読み込みと適用のための機能があり、同じファイルからの設定の再読み込みのリクエストを待ち受けているgoroutineがある。
どちらの仕組みもこの後で説明されている。

### 設定の読み込みと解析

最初の設定が読み込まれたまたはそれ以降の再読み込みが起きた時に、Prometheusは、[`config.LoadFile()`](https://github.com/prometheus/prometheus/blob/v2.3.1/config/config.go#L52-L64)を呼び出し、設定をファイルから読み込み、解析して[`config.Config` structure](https://github.com/prometheus/prometheus/blob/v2.3.1/config/config.go#L133-L145)に変換する。
この構造体は、サーバーとその全てのコンポーネントの設定全体を表す。
これは、設定ファイルの階層構造を反映した下層構造を含んでいる。
各構造体には、[デフォルトの設定](https://github.com/prometheus/prometheus/blob/v2.3.1/config/config.go#L66-L131)およびYAMLから構造体へと変換しさらに有効性を検査したり初期化する`UnmarshalYAML()`メソッドがある。
設定が完全に解析されて検証が終わると、その結果の設定の構造体が返される。


### 再読み込みハンドラー

設定再読み込みハンドラーは、`main()`の中で直接実装されているgoroutineであり、webインターフェースまたは[シグナル`HUP`](https://ja.wikipedia.org/wiki/シグナル (Unix))からの設定再読み込みリクエストを待ち受けている。
再読み込みリクエストを受け取ると、`config.LoadFile()`を使って上述のように設定ファイルをディスクから読み直し、再読み込みをサポートしている全てのコンポーネントに、[`ApplyConfig()`メソッドを呼び出すか、独自の再読み込み関数を呼び出して](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L302-L341)、その結果の構造体`config.Config`を適用する。

## 終了ハンドラー

[終了ハンドラー](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L367-L392)は、`main()`の中で直接実装されているgoroutineであり、webインターフェースまたは[シグナル`TERM`](https://ja.wikipedia.org/wiki/シグナル (Unix))からの終了リクエストを待ち受けている。
終了リクエストを受け取ると、リターンして、[`github.com/oklog/oklog/pkg/group`](https://godoc.org/github.com/oklog/run)で提供されているactor coordinationの機能を通じて、他のPrometheusコンポーネント全てを綺麗に終了させる。

## Scrape discovery manager

scrape discovery managerは、[`discovery.Manager`](https://github.com/prometheus/prometheus/blob/v2.3.1/discovery/manager.go#L73-L89)であり、Prometheusのサービスディスカバリー機能を利用して、Prometheusのメトリクス取得元である対象のリストを継続的に更新する。
監視対象を実際にスクレイプをするscrape managerからは独立しで動作し、[synchronization channel](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L431)を通して[target group](https://github.com/prometheus/prometheus/blob/v2.3.1/discovery/targetgroup/targetgroup.go#L24-L33)の更新を流す。

内部的には、scrape discovery managerは、設定で定義されたサービスディスカバリーの仕組みのインスタンスをそれ自体のgoroutineで動作させる。
例えば、設定ファイルの`scrape_config`が2つの[`kubernetes_sd_config`セクション](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Ckubernetes_sd_config%3E)を定義しているとすると、scrape discovery managerは2つの別々の[`kubernetes.Discovery`](https://github.com/prometheus/prometheus/blob/v2.3.1/discovery/kubernetes/kubernetes.go#L150-L159)のインスタンスを動作させる。
これらの`kubernetes.Discovery`はそれぞれ、インターフェース[`discovery.Discoverer`](https://github.com/prometheus/prometheus/blob/v2.3.1/discovery/manager.go#L41-L55)を実装しており、synchronization channelを通して監視対象の更新をdiscovery managerに送信する。
そうすると、discovery managerは、特定のディスカバリーのインスタンスについての情報でターゲットグループを拡充し、それをscrape managerに転送する。

設定の変更が適用されると、discovery managerは、動作中のディスカバリーの仕組みを全て停止し、新しい設定ファイルで定義されたものとして再起動する。

さらなる詳細については、広範な[サービスディスカバリーの内部についてのドキュメント](https://github.com/prometheus/prometheus/blob/master/discovery/README.md)を参照すること。

## Scrape manager

scrape managerは、[`scrape.Manager`](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/manager.go#L47-L62)であり、検出された監視対象からメトリクスを取得し、得られたサンプルをストレージのサブシステムに転送する。

### ターゲット更新と全体の構造

scrape discovery managerが、`scrape_config`それぞれに対して1つのディスカバリーの仕組みを動作させるのと同様の方法で、scrape managerは`scrape_config`それぞれに対して対応するスクレイプのプールを動作させる。
両者は、複数のリロードとこの2つのコンポーネントにまたがっている`scrape_config`を、設定項目`job_name`を通して特定する（`job_name`は1つの設定ファイルの中でユニークでなければならない）。
discovery managerは、`scrape_config`それぞれに対する監視対象の更新をsynchronization channelを通してscrape managerに送信する。
scrape managerは、[それらの更新を対応するscrape poolに適用する](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/manager.go#L150-L173)。
各scrape poolは、監視対象それぞれのスクレイプのループを実行する。
全体の階層構造は以下のようになる。

* Scrape manager
  * Scrape pool for `scrape_config` 1
    * Scrape loop for target 1
    * Scrape loop for target 2
    * Scrape loop for target 3
    * [...]
    * Scrape loop for target n
  * Scrape pool for `scrape_config` 2
    * [...]
  * Scrape pool for `scrape_config` 3
    * [...]
  * [...]
  * Scrape pool for `scrape_config` n
    * [...]

### ターゲットのラベルとリラベル

scrape managerが、あるスクレイプのプールに対する更新された監視対象のリストをdiscovery managerから受け取ると、そのスクレイプのプールは、デフォルトのターゲットラベル（`job`、`instance`など）を各ターゲットに適用し、最終的にスクレイプすべき監視対象のリストを生成するために[リラベルの設定](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Crelabel_config%3E)を適用する。

### ターゲットのハッシュとスクレイプのタイミング

To spread out scrapes within a scrape pool and in a consistently slotted way across the `scrape_config`'s scrape interval, each target is [hashed by its label set and its final scrape URL](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/target.go#L75-L82). This hash is then used to [choose a deterministic offset](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/target.go#L84-L98) within that interval.
スクレイプのプール内にスクレイプを拡散し、
`scrape_config`のスクレイプ間隔に絶えず
各監視対象は、[ラベル集合と最終的なスクレイプURLでハッシュが取られる](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/target.go#L75-L82)。
このハッシュは、スクレイプ間隔内の[オフセットを選ぶ](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/target.go#L84-L98)ために利用される。

### 監視対象のスクレイプ

最後に、スクレイプのループは、定期的に、HTTPで監視対象をスクレイプし、受信したHTTPレスポンスを[Prometheusのテキストベースのメトリクス出力フォーマット](https://prometheus.io/docs/instrumenting/exposition_formats/)に従ってデコードする。
個々のサンプルそれぞれに[メトリックのリラベル設定](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Cmetric_relabel_configs%3E)を適用し、その結果のサンプルをストレージのサブシステムに送信する。
さらに、複数のスクレイプの実行にまたがる時系列の失効を追跡し保存し、[スクレイプの状態の情報](https://prometheus.io/docs/concepts/jobs_instances/#automatically-generated-labels-and-time-series)（`up`や`scrape_duration_seconds`などのメトリクス）を記録し、ストレージエンジンへの時系列の追加を最適化するためのデータの整理を行う。
1回のスクレイプは、設定されたスクレイプ間隔より時間がかかってはいけないし、スクレイプのタイムアウトはそれが上限になっていることに注意。
これによって、1回のスクレイプが別のスクレイプが始まる前に終わらされるのが確実になる。

## ストレージ

Prometheusは、時系列のサンプルをローカルの時系列データベース（TSDB）に保存し、オプションで、全サンプルのコピーを設定されたリモートのエンドポイントに転送する。
同様に、Prometheusは、ローカルのTSDBおよびオプションでリモートのエンドポイントからデータを読み取る。
ローカルとリモートのストレージのサブシステムについて以下で説明する。

### fanoutストレージ

[fanout storage](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/fanout.go#L27-L32)は、他のコンポーネントによる利用のために、背後にあるローカルおよびリモートのストレージの詳細をプロキシ、抽象化する[`storage.Storage`](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/interface.go#L31-L44)の実装である。
読み込みのためにローカルとリモートの読み込み元からのクエリの結果をマージし、書き込みはローカルとリモートの全ての書き込み先に複製される。
時系列の取り込みを最適化するための機能は様々なので、内部的には、第1の（ローカルの）ストレージとオプションの第2の（リモートの）ストレージを区別する。

現在、ルールはまだ、直接、fanout storageから読み込み、fanout storageへ書き込みをしているが、しばらくすると、デフォルトでルールがローカルデータを読みこむだけになるように変更されるだろう。
これで、ほとんどの場合で短期のデータしか必要のないアラートとレコーディングルールの信頼性が増す。

### ローカルストレージ

Prometheusのローカルディクス上の時系列データベースは、[`github.com/prometheus/tsdb.DB`](https://github.com/prometheus/tsdb/blob/master/db.go#L92-L117)に対する[軽量ラッパー](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/tsdb/tsdb.go#L102-L106)である。
このラッパーは、Prometheusサーバー環境でTSDBを利用するための若干のインターフェースの調整だけをし、[`storage.Storage`インターフェース](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/interface.go#L31-L44)を実装する。
[ローカルストレージのドキュメント](https://prometheus.io/docs/prometheus/latest/storage/)にTSDBのディスク上のレイアウトさらなる詳細がある。

### リモートストレージ

リモートストレージは、[`remote.Storage`](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/remote/storage.go#L31-L44)であり、[`storage.Storage`インターフェース](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/interface.go#L31-L44)を実装し、リモートの読み込みと書き込みのエンドポイントとの連結を担っている。

設定ファイルの[`remote_write`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Cremote_write%3E)それぞれに対して、remote storageは1つの[`remote.QueueManager`](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/remote/queue_manager.go#L141-L161)を作成し、動作させる。
次に、`remote.QueueManager`は、サンプルをキューに入れ、リモートの書き込みエンドポイントに送信する。
各キューマネージャーは、現在と過去の負荷の観察に基づいて動的に決まる数のシャードを動作させることで、リモートエンドポイントへの書き込みを並列化する。
設定の再読み込みが適用されると、全てのリモートストレージのキューはシャットダウンされ、新しいキューが作成される。

設定ファイルの[`remote_read`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#%3Cremote_read%3E)それぞれに対して、remote storageは1つの[reader client](https://github.com/prometheus/prometheus/blob/v2.3.1/storage/remote/storage.go#L96-L118)を作成し、各リモートソースからの結果はマージされる。

## PromQLエンジン

[PromQLエンジン](https://github.com/prometheus/prometheus/blob/v2.3.1/promql/engine.go#L164-L171)は、[PromQL expression queries](https://prometheus.io/docs/prometheus/latest/querying/basics/)のPrometheusの時系列データベースに対して評価する役割を担っている。
このエンジンはそれ自身アクターのgoroutineとして動作せず、webインターフェースとルールマネージャーからライブラリとして利用される。
PromQLの評価は、複数の段階を経る。クエリが作成されると、その式は抽象的な構文木へとパースされ、実行可能なクエリとなる。
後続の実行段階では、背後のストレージから得られる必要な時系列を検索し、そのためのイテレーターを作成する。
時系列の大量データの実際の取得は、（少なくともローカルのTSDBの場合は）評価時に行われる。
式の評価は、PromQLの式の型（ほとんどの場合はinstant vectorまたはrange vector）を返す。

## Rule manager

rule managerは、[`rules.Manager`](https://github.com/prometheus/prometheus/blob/v2.3.1/rules/manager.go#L410-L418)であり、（設定ファイルの`evaluation_interval`で設定されているように）定期的なレコーディングルールとアラートルールの評価をする役割を担っている。
各繰り返しで、PromQLを利用して全てのルールを評価し、結果の時系列をストレージに書き込む。

アラートルールに関しては、rule managerは、以下のように、各繰り返しでいくつかのことを行う。

- pendingまたはfiringのアラートに対して時系列`ALERTS{alertname="<alertname>", <alert labels>}`を保存する
- アラートルールの`for`の時間に基づいて、pendingからfiringにいつ遷移するか決めるためにactiveなアラートのライフサイクルの状態を追跡する
- activeなアラートそれぞれに対してアラートルールのラベルとアノテーションのテンプレートを展開する
- firingなアラートをnotifierに送り（下記参照）、解消したアラートは15分間送り続ける

## Notifier

notifierは、[`notifier.Manager`](https://github.com/prometheus/prometheus/blob/v2.3.1/notifier/notifier.go#L104-L119)であり、rule managerが生成したアラートを`Send()`メソッドで受け取り、キューに入れ、設定されたAlertmanagerインスタンス全てに送信する。
notifierは、アラートの生成とアラートのAlertmanagerへの送信（失敗したり時間がかかったりする）を分離する役割を果たす。

## Notifier discovery

notifier discovery managerは、[`discovery.Manager`](https://github.com/prometheus/prometheus/blob/v2.3.1/discovery/manager.go#L73-L89)であり、Prometheusのサービスディスカバリーの機能を利用して、notifierがアラートを送るAlertmanagerのインスタンスのリストを検索し継続的に更新する。
notifier discovery managerは、notifier managerとは独立して動作し、[synchronization channel](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L587)を通してtarget groupの更新を次々にnotifier managerに送信する。

内部的には、scrape discovery managerと同様な仕組みで動作する。

## Web UI and API

Prometheusは、デフォルトでは`9090`ポートでweb UIとAPIを提供する。
web UIは`/`で利用可能であり、クエリを実行したり、activeなアラートを調査したり、その他のPrometheusサーバーの状態を理解するために人間が利用するインターフェースとして機能する。

web APIは、`/api/v1`で提供され、[クエリ、メタデータ、サーバーの状態をプログラムで調べる](https://prometheus.io/docs/prometheus/latest/querying/api/)ことを可能にしている。

コンソールテンプレートは、TSDBデータにアクセスできるユーザー定義のHTMLテンプレートをPrometheusが提供出来るようにするものであり、設定されていれば`/consoles`で利用可能である。
