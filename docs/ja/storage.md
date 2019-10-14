---
title: ストレージ
sort_rank: 5
---

# <span class="anchor-text-supplement">Storage</span>ストレージ

Prometheusは、ローカルのディスク上に時系列データベースを持っているが、オプションでリモートのストレージシステムとも連携する。

## <span class="anchor-text-supplement">Local storage</span>ローカルストレージ

Prometheusのローカル時系列データーベースは、時系列データを独自フォーマットでディスク上に保存する。

### <span class="anchor-text-supplement">On-disk layout</span>ディクス上の構成

取り込まれた値は、2時間のブロックにまとめられる。
2時間のブロックそれぞれは、1つ以上のチャンクファイル、メタデータファイル、インデックスファイルを含むディレクトリから成る。
チャンクファイルは、その時間の時系列の全ての値を含む。
インデックスファイルは、メトリック名とラベルからチャンクファイル内の時系列へのインデックスを含む。
時系列がAPIを通して削除されると、削除レコードが別のtombstoneファイルに保存される（データはチャンクファイルから即時に削除されない）。

現在取り込みつつある値のブロックは、メモリに保持され、完全に永続化はまだされていない。
クラッシュに対して安全にするためにwrite-ahead-log（WAL）が利用される。
write-ahead logは、Prometheusサーバーがクラッシュ後に再起動した際にやり直しが出来る。
Write-ahead logファイルはディレクトリ`wal`に128MBセグメントで保存される。
これらのファイルは、まだコンパクションされていない生データを含むので、通常のブロックファイルよりかなり大きい。
Prometheusは、少なくとも3つのwrite-ahead logを保持するが、トラフィックの多いサーバーでは、3つより多いWALファイルが見れらる。
なぜなら、少なくとも2時間相当の生データを保持しなければならないからである。

Prometheusサーバーのデータディレクトリの構成は、以下のようなものになるだろう。

```
./data
├── 01BKGV7JBM69T2G1BGBGM6KB12
│   └── meta.json
├── 01BKGTZQ1SYQJTR4PB43C8PD98
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
├── 01BKGTZQ1HHWHV8FBJXW1Y3W0K
│   └── meta.json
├── 01BKGV7JC0RY8A6MACW02A2PJD
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
└── wal
    ├── 00000002
    └── checkpoint.000001
```

ローカルストレージの制限は、クラスタ化されたり複製されたりしないことである。
したがって、自由にスケールしないし、ディスクやノードの停止に耐えられず、短期の時間枠の最近のデータとして扱われるべきである。
ただし、耐久性の要件が厳格でなければ、ローカルストレージに何年ものデータを保存し続けることが出来ることもある。

ファイルフォーマットのさらなる詳細は、[TSDB format](https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/README.md)を参照すること。

## <span class="anchor-text-supplement">Compaction</span>コンパクション

最初の2時間のブロックは、最終的にバックグラウンドでより長いブロックへとコンパクションされる。

コンパクションは、保持期間の10%または21日間のいずれか小さい方より大きなブロックを作るだろう。

## <span class="anchor-text-supplement">Operational aspects</span>運用に関わる側面

Prometheusには、ローカルストレージを設定するいくつかのフラグがある。
特に重要なものは、以下の通り。

* `--storage.tsdb.path`: Prometheusがどこにデータベースを書き込むかを決める。デフォルトは、`data/`
* `--storage.tsdb.retention.time`: 古いデータをいつ削除するかを決める。デフォルトは、15d。storage.tsdb.retentionがデフォルト以外にセットされていたら、上書きする
* `--storage.tsdb.retention.size`: [EXPERIMENTAL] ストレージのブロックが利用できる最大バイト数を決める。WALはかなりのサイズに成るが、これには含まれていないことに注意。最も古いデータが最初に削除される。デフォルトは0、つまり無効化されている。このフラグは、実験的であり、将来のリリースで変更される可能性がある。サポートされている単位は、KB、MB、GB、PB. 例: "512MB"
* `--storage.tsdb.retention`: このフラグは非推奨である。`storage.tsdb.retention.time`を利用するのが望ましい。
* `--storage.tsdb.wal-compression`: このフラグは、write-ahead log（WAL）の圧縮を有効にする。データによっては、CPU負荷を少しだけ余分にかけるだけでWALのサイズが半分になることが期待できる。もし、このフラグを有効にしてその後にPrometheusを2.11.0よりもダウングレードすると、WALは読み込めないので消す必要が出てくることに注意すること。

Prometheusは、平均的に、1つの観測値につき大体1〜2バイトしか使わない。
したがって、Prometheusサーバーのキャパシティを計画するには、以下の式を利用することができる。

```
needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample
```

*訳注: 必要なディスクスペース = 保持期間（秒） * 秒間サンプル採取数 * サンプルのバイト数*

秒間サンプル採取数を調整するために、取得する時系列の数を減らす（ターゲットを少なくしたりターゲットあたりの時系列を少なくする）か取得間隔を長くすることができる。
ただし、1つの時系列の中で観測値の圧縮があるので、時系列の数を減らす方が効果的である可能性が高い。

もし、ローカルストレージが壊れたら、理由が何であれ、Prometheusを停止してストレージディレクトリ全体を消すのが最善策である。
POSIX互換でないファイルシステムは、Prometheusのローカルストレージでサポートされていない。
復元の可能性のない破壊が起きる可能性がある。
NFSは、POSIXの可能性はあるが、ほとんどの実装はPOSIXではない。
問題を解決するために、個々のブロックディレクトリを削除することも可能だが、ブロックディレクトリ毎に約2時間相当のデータを失うことになる。
Prometheusのローカルストレージは、長期保存に耐えられるようには意図されていないのである。

時間と容量の両方の保持ポリシーが指定されている場合、先に引っかかった条件がその時点で使われる。

有効期限切れしたブロックの削除は、バックグラウンドのスケジュールで実行される。有効期限切れしたブロックを削除するには2時間かかる可能性がある。有効期限切れしたブロックは、削除される前に完全に有効期限切れしている必要がある。

## <span class="anchor-text-supplement">Remote storage integrations</span>リモートストレージ連携

Prometheusのローカルストレージは、スケーラビリティと耐久性の点で、1つのノードに制限されている。
Prometheus自体でクラスタ化されたストレージに対処する代わりに、Prometheusにはリモートストレージシステムと連携するインターフェースがある。

### <span class="anchor-text-supplement">Overview</span>概要

Prometheusは、以下の2つの方法でリモートストレージシステムと連携する。

* Prometheusは、取り込んだ値を標準化されたフォーマットでリモートURLに書き込むことができる
* Prometheusは、標準化されたフォーマットでリモートURLから値を読み取ることができる

![Remote read and write architecture](images/remote_integrations.png)

読み書きのプロトコルはどちらもsnappyで圧縮されたプロトコルバッファをHTTP越しに利用する。
これらのプロトコルはまだ安定したAPIと見なされておらず、将来的には、Prometheusとリモートストレージの間の経路が全てHTTP/2をサポートしたと仮定できるようになれば、gRPC over HTTP/2に変わるかもしれない。

Prometheusのリモートストレージ連携の設定に関する詳細は、Prometheus設定ドキュメントの[remote write](http://it-engineer.hateblo.jp/entry/2019/05/02/154006)と[remote read](http://it-engineer.hateblo.jp/entry/2019/05/02/153031)の部分を参照すること。

リクエストとレスポンスのメッセージに関する詳細は、[リモートストレージプロトコルバッファの定義](https://github.com/prometheus/prometheus/blob/master/prompb/remote.proto)を参照すること。

読み込む行程では、Prometheusはラベルセレクターの集合と時間幅に対する生の時系列データを取得するだけであることに注意すること。
生データに対するPromQLの全ての評価はPrometheus自体の中で起きる。
つまり、必要なデータは全てクエリを発行しているPrometheusサーバーにまず読み込まれてその後Prometheusサーバで処理される必要があるので、remote readのクエリにはスケーラビリティの制限がある。
しかし、完全に分散されたPromQLの評価をサポートすることはしばらくの間は不可能だと考えられている。

### <span class="anchor-text-supplement">Existing integrations</span>既存の連携

リモートストレージとの既存の連携についてさらに学ぶには、[インテグレーション](https://prometheus.io/ja/docs/operating/integrations/#remote-endpoints-and-storage)のドキュメントを参照すること。