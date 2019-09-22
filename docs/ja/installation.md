---
title: インストール
sort_rank: 2
---

# インストール

## コンパイル済みのバイナリの利用

ほとんどの公式Prometheusコンポーネントに関して、コンパイル済みのバイナリが提供されている。
全ての利用可能なバージョンのリストは、[ダウンロードページ](https://prometheus.io/download)を調べること。

## ソースから

Prometheusコンポーネントをソースからビルドするには、各リポジトリの`Makefile`ターゲットを参照すること。

## Dockerの利用

Prometheusサービスは全て、[Quay.io](https://quay.io/repository/prometheus/prometheus)や[Docker Hub](https://hub.docker.com/r/prom/prometheus/)でDockerイメージとして利用可能である。

DockerでPrometheusを実行するには単に`docker run -p 9090:9090 prom/prometheus`を実行すれば良い。
これで、サンプルの設定でPrometheusを起動し、ポート9090でメトリクスを出力する。

Prometheusイメージは、実際のメトリクスを保存するためにボリュームを利用する。
Prometheusのアップグレード時にデータ管理を簡単にするために、本番デプロイには[Data Volume Container](https://docs.docker.com/storage/volumes/)を利用することを強く推奨する。

自分自身の設定を与えるには、いくつか選択肢がある。ここでは2つの例を示す。

### Volumes & bind-mount

以下のコマンドを実行することで、そのホストから自分の`prometheus.yml`をbind-mountする。

```bash
docker run -p 9090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml \
       prom/prometheus
```

あるいは、設定のための追加のボリュームを利用する。

```bash
docker run -p 9090:9090 -v /prometheus-data \
       prom/prometheus --config.file=/prometheus-data/prometheus.yml
```

### 独自イメージ

ファイルを管理してそれをbind-mountするのを避けるために、設定をイメージに焼き付けることが出来る。
これは、設定自体がかなり静的であり、全ての環境に渡って同じならば、うまく行く。

Prometheusの設定を含む新しいディレクトリと以下のような`Dockerfile`を作成する。

```Dockerfile
FROM prom/prometheus
ADD prometheus.yml /etc/prometheus/
```

これをビルド、実行する。

```bash
docker build -t my-prometheus .
docker run -p 9090:9090 my-prometheus
```

より高度な選択肢としては、何らかのツールで設定を起動時に動的に与えるか、デーモンで設定を定期的に更新する。

## 設定管理システムの利用

設定管理システムが好ましいのであれば、以下のサードパーティからの貢献に興味が湧くだろう。

### Ansible

* [Cloud Alchemy/ansible-prometheus](https://github.com/cloudalchemy/ansible-prometheus)

### Chef

* [rayrod2030/chef-prometheus](https://github.com/rayrod2030/chef-prometheus)

### Puppet

* [puppet/prometheus](https://forge.puppet.com/puppet/prometheus)

### SaltStack

* [bechtoldt/saltstack-prometheus-formula](https://github.com/bechtoldt/saltstack-prometheus-formula)
