---
title: 設定
sort_rank: 1
---

# 設定

Prometheusは、コマンドラインフラグと設定ファイルを通して設定される。
コマンドラインフラグは、不変のシステムパラメーター（ストレージの場所、ディスクやメモリに保持するデータ量など）を設定し、設定ファイルは、どの[ルールファイル]((recording_rules.md#configuring-rules))を読み込むかに加えて、スクレイピングの[ジョブとインスタンス](https://prometheus.io/docs/concepts/jobs_instances/)に関する全てを定義する。

利用可能な全てのコマンドラインフラグを見るには、`./prometheus -h`を実行する。

Prometheusは、実行時に設定を読み込み直すことが出来る。
もし、新しい設定が正しい形式でなければ、変更は適用されない。
設定を読み込み直させるには、Prometheusプロセスに`SIGHUP`を送るか、（`--web.enable-lifecycle`フラグが有効な場合）エンドポイント`/-/reload`にHTTP POSTリクエストを送る。

## 設定ファイル

どの設定ファイルを読み込むか指定するには、フラグ`--config.file`を使う。

設定ファイルは、以下に記すスキームで定義された[YAML形式](https://ja.wikipedia.org/wiki/YAML)で書かれる。
ブラケットは、パラメーターがオプションであることを表す。
リストでないパラメーターは、指定されたデフォルト値にセットされる。

一般的なプレースホルダーは以下の通り。

* `<boolean>`: 値として`true`または`false`を取ることが出来るブーリアン
* `<duration>`: 正規表現`[0-9]+(ms|[smhdwy])`にマッチする時間幅
* `<labelname>`: 正規表現`[a-zA-Z_][a-zA-Z0-9_]*`にマッチする文字列
* `<labelvalue>`: Unicodeの文字列
* `<filename>`: ワーキングディレクトリ内のパス
* `<host>`: ホスト名またはIP、オプションでポート番号から成る文字列
* `<path>`: URLパス
* `<scheme>`: 値として`http`または`https`を取ることが出来る文字列
* `<string>`: 文字列
* `<secret>`: パスワードのような秘密の文字列
* `<tmpl_string>`: 利用前にテンプレートで展開される文字列

そのほかのプレースホルダーは、個別に定義される。

[ここ](https://github.com/prometheus/prometheus/blob/release-2.9/config/testdata/conf.good.yml)でサンプルファイルを見ることが出来る。

グローバルな設定は、他の全ての部分で有効なパラメーターを指定する。
これらは、他の部分での設定のデフォルト値にもなる。

```yaml
global:
  # デフォルトでターゲットをスクレイプする頻度
  [ scrape_interval: <duration> | default = 1m ]

  # スクレイプのリクエストがタイムアウトするまでの時間
  [ scrape_timeout: <duration> | default = 10s ]

  # ルールを評価する頻度
  [ evaluation_interval: <duration> | default = 1m ]

  # 外部システム（federation、remote storage、Alertmanager）と通信するときに
  # 全ての時系列やアラートに追加するラベル
  external_labels:
    [ <labelname>: <labelvalue> ... ]

# ルールファイルはグロブのリストを指定する。
# マッチした全てのファイルからルールとアラートが読み込まれる
rule_files:
  [ - <filepath_glob> ... ]

# スクレイプの設定のリスト
scrape_configs:
  [ - <scrape_config> ... ]

# alertingはAlertmanagerに関する設定をする
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# remote write機能に関する設定
remote_write:
  [ - <remote_write> ... ]

# remote read機能に関する設定
remote_read:
  [ - <remote_read> ... ]
```

### `<scrape_config>`

`scrape_config`セクションは、監視対象とそれらをどのようにscrapeするかを表すパラメーターの集合である。
一般的なケースでは、一つのscrapeの設定が一つのジョブを指定する。高度な設定ではそうでない場合もある。

監視対象は、`static_configs`によって静的に設定されることも、サービスディスカバリーの仕組みによって動的に設定されることもある。

`relabel_configs`によって、スクレイプする前に任意の対象・ラベルを変更することが可能である。

```yaml
# The job name assigned to scraped metrics by default.
job_name: <job_name>

# How frequently to scrape targets from this job.
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# Per-scrape timeout when scraping this job.
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# The HTTP resource path on which to fetch metrics from targets.
[ metrics_path: <path> | default = /metrics ]

# honor_labelsは、取得したデータにすでにあるラベルとPrometheusが付与しようとしている
# サーバーサイドのラベル（jobやinstance、手動で設定されたラベル、サービスディスカバリー
# で生成されたラベル）の衝突を制御する
#
# honor_labelsがtrueの場合、取得したデータのラベルの値を保持し、衝突しているサーバーサ
# イドのラベルを無視することで衝突を解決する
#
# honor_labelsがfalseの場合、取得したデータにある衝突しているラベルを
# "exported_<original-label>"（例えばexported_instance、exported_job）という名前に変更
# してからサーバーサイドのラベルを付与することで衝突を解決する。
#
# honor_labelsをtrueに設定することは、フェデレーションやPushgatewayのスクレイプのような、
# 対象の全てのラベルを保持しなければいけないユースケースで便利である。
#
# グローバルに設定された"external_labels"は、この設定に影響されないことに注意すること。
# 外部システムとのやりとりでは、それらは必ず時系列データにそのラベルがまだない場合にのみ適用され、
# そうでない場合は無視される。
[ honor_labels: <boolean> | default = false ]

# honor_timestampsは、スクレイプしたデータに存在するタイムスタンプをPrometheusが重んじるかを制御する。
#
# honor_timestampsがtrueに設定されている場合、監視対象によって出力されたメトリクスのタイムスタンプが利用される。
#
# honor_timestampsがfalseに設定されている場合、監視対象によって出力されたメトリクスのタイムスタンプは無視される。
[ honor_timestamps: <boolean> | default = true ]

# Configures the protocol scheme used for requests.
[ scheme: <scheme> | default = http ]

# Optional HTTP URL parameters.
params:
  [ <string>: [<string>, ...] ]

# 設定されたusernameとpasswordでscrapeのリクエストの`Authorization`ヘッダを毎回セットする。
# passwordとpassword_fileは相互排他的である。
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 設定された署名なしトークン (Bearer Token)でscrapeのリクエストの`Authorization`ヘッダを
# 毎回セットする。`bearer_token_file`と相互排他的である。
[ bearer_token: <secret> ]

# 設定ファイルから読み込んだ署名なしトークン (Bearer Token)でscrapeのリクエストの
# `Authorization`ヘッダを毎回セットする。`bearer_token`と相互排他的である。
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the scrape request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# List of Azure service discovery configurations.
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# List of Consul service discovery configurations.
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# List of DNS service discovery configurations.
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# List of EC2 service discovery configurations.
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# List of OpenStack service discovery configurations.
openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

# List of file service discovery configurations.
file_sd_configs:
  [ - <file_sd_config> ... ]

# List of GCE service discovery configurations.
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# List of Kubernetes service discovery configurations.
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# List of Marathon service discovery configurations.
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# List of AirBnB's Nerve service discovery configurations.
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# List of Zookeeper Serverset service discovery configurations.
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# List of Triton service discovery configurations.
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# List of labeled statically configured targets for this job.
static_configs:
  [ - <static_config> ... ]

# List of target relabel configurations.
relabel_configs:
  [ - <relabel_config> ... ]

# List of metric relabel configurations.
metric_relabel_configs:
  [ - <relabel_config> ... ]

# スクレイプ毎に受け付けるサンプル数の限界値。もしリラベルした後にこの値より多いサンプルに
# なったらスクレイプ全体が失敗として扱われる。0は制限なしを意味する
[ sample_limit: <int> | default = 0 ]
```

ここで、`<job_name>`は全てのスクレイプの設定にまたがってユニークでなければならない。

### `<tls_config>`

`tls_config`によって、TLS接続の設定ができる。

```yaml
# APIサーバーを証明するためのCA証明書
[ ca_file: <filename> ]

# クライアント証明書を認証するための証明書と鍵ファイル
[ cert_file: <filename> ]
[ key_file: <filename> ]

# Server Name Indicationのためのサーバー名
# https://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# サーバー証明書の検証の無効化
[ insecure_skip_verify: <boolean> ]
```

### `<azure_sd_config>`

Azure SD configurations allow retrieving scrape targets from Azure VMs.

The following meta labels are available on targets during relabeling:

* `__meta_azure_machine_id`: the machine ID
* `__meta_azure_machine_location`: the location the machine runs in
* `__meta_azure_machine_name`: the machine name
* `__meta_azure_machine_os_type`: the machine operating system
* `__meta_azure_machine_public_ip`: the machine's public IP if it exists
* `__meta_azure_machine_private_ip`: the machine's private IP
* `__meta_azure_machine_resource_group`: the machine's resource group
* `__meta_azure_machine_tag_<tagname>`: each tag value of the machine
* `__meta_azure_machine_scale_set`: the name of the scale set which the vm is part of (this value is only set if you are using a [scale set](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/))
* `__meta_azure_subscription_id`: the subscription ID
* `__meta_azure_tenant_id`: the tenant ID

See below for the configuration options for Azure discovery:

```yaml
# The information to access the Azure API.
# The Azure environment.
[ environment: <string> | default = AzurePublicCloud ]

# The authentication method, either OAuth or ManagedIdentity.
# See https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview
[ authentication_method: <string> | default = OAuth]
# The subscription ID. Always required.
subscription_id: <string>
# Optional tenant ID. Only required with authentication_method OAuth.
[ tenant_id: <string> ]
# Optional client ID. Only required with authentication_method OAuth.
[ client_id: <string> ]
# Optional client secret. Only required with authentication_method OAuth.
[ client_secret: <secret> ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 300s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]
```

### `<consul_sd_config>`

Consul SD configurations allow retrieving scrape targets from [Consul's](https://www.consul.io)
Catalog API.

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_consul_address`: the address of the target
* `__meta_consul_dc`: the datacenter name for the target
* `__meta_consul_tagged_address_<key>`: each node tagged address key value of the target
* `__meta_consul_metadata_<key>`: each node metadata key value of the target
* `__meta_consul_node`: the node name defined for the target
* `__meta_consul_service_address`: the service address of the target
* `__meta_consul_service_id`: the service ID of the target
* `__meta_consul_service_metadata_<key>`: each service metadata key value of the target
* `__meta_consul_service_port`: the service port of the target
* `__meta_consul_service`: the name of the service the target belongs to
* `__meta_consul_tags`: the list of tags of the target joined by the tag separator

```yaml
# The information to access the Consul API. It is to be defined
# as the Consul documentation requires.
[ server: <host> | default = "localhost:8500" ]
[ token: <secret> ]
[ datacenter: <string> ]
[ scheme: <string> | default = "http" ]
[ username: <string> ]
[ password: <secret> ]

tls_config:
  [ <tls_config> ]

# A list of services for which targets are retrieved. If omitted, all services
# are scraped.
services:
  [ - <string> ]

# See https://www.consul.io/api/catalog.html#list-nodes-for-service to know more
# about the possible filters that can be used.

# An optional list of tags used to filter nodes for a given service. Services must contain all tags in the list.
tags:
  [ - <string> ]

# Node metadata used to filter nodes for a given service.
[ node_meta:
  [ <name>: <value> ... ] ]

# The string by which Consul tags are joined into the tag label.
[ tag_separator: <string> | default = , ]

# Allow stale Consul results (see https://www.consul.io/api/features/consistency.html). Will reduce load on Consul.
[ allow_stale: <bool> ]

# The time after which the provided names are refreshed.
# On large setup it might be a good idea to increase this value because the catalog will change all the time.
[ refresh_interval: <duration> | default = 30s ]
```

Note that the IP number and port used to scrape the targets is assembled as
`<__meta_consul_address>:<__meta_consul_service_port>`. However, in some
Consul setups, the relevant address is in `__meta_consul_service_address`.
In those cases, you can use the [relabel](#relabel_config)
feature to replace the special `__address__` label.

The [relabeling phase](#relabel_config) is the preferred and more powerful
way to filter services or nodes for a service based on arbitrary labels. For
users with thousands of services it can be more efficient to use the Consul API
directly which has basic support for filtering nodes (currently by node
metadata and a single tag).

### `<dns_sd_config>`

DNSベースのサービスディスカバリーの設定によって、ターゲットのリストを検出するために検索されるDNSドメイン名の集合を指定することができる。
通信するDNSサーバーは、`/etc/resolv.conf`から読み込まれる。

このサービスディスカバリーの方法は、基本的なDNSのA、AAAA、SRVレコードの検索をサポートするだけで、[RFC6763](https://tools.ietf.org/html/rfc6763)で定義されているDNS-SDの方法はサポートしていない。

[リラベル](#relabel_config)の過程で、各ターゲットでメタラベル`__meta_dns_name`が利用可能で、検出された対象を生成したレコード名にセットされる。

```yaml
# 検索されるドメイン名のリスト
names:
  [ - <domain_name> ]

# 実行するDNSクエリのタイプ
[ type: <query_type> | default = 'SRV' ]

# クエリタイプがSRV出ない場合に使われるポート番号
[ port: <number>]

# 与えられた名前がリフレッシュされるまでの時間
[ refresh_interval: <duration> | default = 30s ]
```

`<domain_name>`は、正当なDNSドメイン名である。
`<query_type>`は、`SRV`、`A`、`AAAA`のいずれかである。

### `<ec2_sd_config>`

EC2 SD configurations allow retrieving scrape targets from AWS EC2
instances. The private IP address is used by default, but may be changed to
the public IP address with relabeling.

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_ec2_availability_zone`: the availability zone in which the instance is running
* `__meta_ec2_instance_id`: the EC2 instance ID
* `__meta_ec2_instance_state`: the state of the EC2 instance
* `__meta_ec2_instance_type`: the type of the EC2 instance
* `__meta_ec2_owner_id`: the ID of the AWS account that owns the EC2 instance
* `__meta_ec2_platform`: the Operating System platform, set to 'windows' on Windows servers, absent otherwise
* `__meta_ec2_primary_subnet_id`: the subnet ID of the primary network interface, if available
* `__meta_ec2_private_dns_name`: the private DNS name of the instance, if available
* `__meta_ec2_private_ip`: the private IP address of the instance, if present
* `__meta_ec2_public_dns_name`: the public DNS name of the instance, if available
* `__meta_ec2_public_ip`: the public IP address of the instance, if available
* `__meta_ec2_subnet_id`: comma separated list of subnets IDs in which the instance is running, if available
* `__meta_ec2_tag_<tagkey>`: each tag value of the instance
* `__meta_ec2_vpc_id`: the ID of the VPC in which the instance is running, if available

See below for the configuration options for EC2 discovery:

```yaml
# The information to access the EC2 API.

# The AWS region. If blank, the region from the instance metadata is used.
[ region: <string> ]

# Custom endpoint to be used.
[ endpoint: <string> ]

# The AWS API keys. If blank, the environment variables `AWS_ACCESS_KEY_ID`
# and `AWS_SECRET_ACCESS_KEY` are used.
[ access_key: <string> ]
[ secret_key: <secret> ]
# Named AWS profile used to connect to the API.
[ profile: <string> ]

# AWS Role ARN, an alternative to using AWS API keys.
[ role_arn: <string> ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# Filters can be used optionally to filter the instance list by other criteria.
# Available filter criteria can be found here:
# https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html
# Filter API documentation: https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_Filter.html
filters:
  [ - name: <string>
      values: <string>, [...] ]
```

The [relabeling phase](#relabel_config) is the preferred and more powerful
way to filter targets based on arbitrary labels. For users with thousands of
instances it can be more efficient to use the EC2 API directly which has
support for filtering instances.

### `<openstack_sd_config>`

OpenStack SD configurations allow retrieving scrape targets from OpenStack Nova
instances.

One of the following `<openstack_role>` types can be configured to discover targets:

#### `hypervisor`

The `hypervisor` role discovers one target per Nova hypervisor node. The target
address defaults to the `host_ip` attribute of the hypervisor.

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_openstack_hypervisor_host_ip`: the hypervisor node's IP address.
* `__meta_openstack_hypervisor_name`: the hypervisor node's name.
* `__meta_openstack_hypervisor_state`: the hypervisor node's state.
* `__meta_openstack_hypervisor_status`: the hypervisor node's status.
* `__meta_openstack_hypervisor_type`: the hypervisor node's type.

#### `instance`

The `instance` role discovers one target per network interface of Nova
instance. The target address defaults to the private IP address of the network
interface.

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_openstack_address_pool`: the pool of the private IP.
* `__meta_openstack_instance_flavor`: the flavor of the OpenStack instance.
* `__meta_openstack_instance_id`: the OpenStack instance ID.
* `__meta_openstack_instance_name`: the OpenStack instance name.
* `__meta_openstack_instance_status`: the status of the OpenStack instance.
* `__meta_openstack_private_ip`: the private IP of the OpenStack instance.
* `__meta_openstack_project_id`: the project (tenant) owning this instance.
* `__meta_openstack_public_ip`: the public IP of the OpenStack instance.
* `__meta_openstack_tag_<tagkey>`: each tag value of the instance.
* `__meta_openstack_user_id`: the user account owning the tenant.

See below for the configuration options for OpenStack discovery:

```yaml
# The information to access the OpenStack API.

# The OpenStack role of entities that should be discovered.
role: <openstack_role>

# The OpenStack Region.
region: <string>

# identity_endpoint specifies the HTTP endpoint that is required to work with
# the Identity API of the appropriate version. While it's ultimately needed by
# all of the identity services, it will often be populated by a provider-level
# function.
[ identity_endpoint: <string> ]

# username is required if using Identity V2 API. Consult with your provider's
# control panel to discover your account's username. In Identity V3, either
# userid or a combination of username and domain_id or domain_name are needed.
[ username: <string> ]
[ userid: <string> ]

# password for the Identity V2 and V3 APIs. Consult with your provider's
# control panel to discover your account's preferred method of authentication.
[ password: <secret> ]

# At most one of domain_id and domain_name must be provided if using username
# with Identity V3. Otherwise, either are optional.
[ domain_name: <string> ]
[ domain_id: <string> ]

# The project_id and project_name fields are optional for the Identity V2 API.
# Some providers allow you to specify a project_name instead of the project_id.
# Some require both. Your provider's authentication policies will determine
# how these fields influence authentication.
[ project_name: <string> ]
[ project_id: <string> ]

# The application_credential_id or application_credential_name fields are
# required if using an application credential to authenticate. Some providers
# allow you to create an application credential to authenticate rather than a
# password.
[ application_credential_name: <string> ]
[ application_credential_id: <string> ]

# The application_credential_secret field is required if using an application
# credential to authenticate.
[ application_credential_secret: <secret> ]

# Whether the service discovery should list all instances for all projects.
# It is only relevant for the 'instance' role and usually requires admin permissions.
[ all_tenants: <boolean> | default: false ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# TLS configuration.
tls_config:
  [ <tls_config> ]
```

### `<file_sd_config>`

ファイルベースのサービスディスカバリーは、静的な対象を設定するためのより一般的な方法を提供し、また
独自のサービスディスカバリーの仕組みを組み込むためのインターフェースとして機能する。

ファイルベースのサービスディスカバリーは、`<static_config>`を0個以上含むファイルの集合を読み込む。
ディスクをウォッチしているので、定義された全てのファイルの変更は即時に検知・適用される。
ファイルはYAMLまたはJSONで記述することが可能である。
正しい形式で書かれた対象グループになる変更のみが適用される。

JSONファイルは、下記のフォーマットで`static_config`のリストを含んでいなければいけない。

```yaml
[
  {
    "targets": [ "<host>", ... ],
    "labels": {
      "<labelname>": "<labelvalue>", ...
    }
  },
  ...
]
```

指定された`refresh_interval`での定期的な再読み込みもされる。

各監視対象は、[リラベルの過程](#relabel_config)でメタラベル`__meta_filepath`を持つ。
その値は、その監視対象が抽出されたファイルパスがセットされる。

この検出の仕組みとの[連携システムの一覧](https://prometheus.io/ja/docs/operating/integrations/#file-service-discovery)がある。

```yaml
# 対象グループを抽出するファイルのパターン
files:
  [ - <filename_pattern> ... ]

# ファイルを再読み込みする間隔
[ refresh_interval: <duration> | default = 5m ]
```

`<filename_pattern>`には、`.json`、`.yml`、`.yaml`で終わるパスを書くことが可能である。
パスの最後の部分は一つの`*`を含んでよい。これによって、任意の文字列の連続（例 `my/path/tg_*.json`）にマッチする。

### `<gce_sd_config>`

[GCE](https://cloud.google.com/compute/) SD configurations allow retrieving scrape targets from GCP GCE instances.
The private IP address is used by default, but may be changed to the public IP
address with relabeling.

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_gce_instance_id`: the numeric id of the instance
* `__meta_gce_instance_name`: the name of the instance
* `__meta_gce_label_<name>`: each GCE label of the instance
* `__meta_gce_machine_type`: full or partial URL of the machine type of the instance
* `__meta_gce_metadata_<name>`: each metadata item of the instance
* `__meta_gce_network`: the network URL of the instance
* `__meta_gce_private_ip`: the private IP address of the instance
* `__meta_gce_project`: the GCP project in which the instance is running
* `__meta_gce_public_ip`: the public IP address of the instance, if present
* `__meta_gce_subnetwork`: the subnetwork URL of the instance
* `__meta_gce_tags`: comma separated list of instance tags
* `__meta_gce_zone`: the GCE zone URL in which the instance is running

See below for the configuration options for GCE discovery:

```yaml
# The information to access the GCE API.

# The GCP Project
project: <string>

# The zone of the scrape targets. If you need multiple zones use multiple
# gce_sd_configs.
zone: <string>

# Filter can be used optionally to filter the instance list by other criteria
# Syntax of this filter string is described here in the filter query parameter section:
# https://cloud.google.com/compute/docs/reference/latest/instances/list
[ filter: <string> ]

# Refresh interval to re-read the instance list
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# The tag separator is used to separate the tags on concatenation
[ tag_separator: <string> | default = , ]
```

Credentials are discovered by the Google Cloud SDK default client by looking
in the following places, preferring the first location found:

1. a JSON file specified by the `GOOGLE_APPLICATION_CREDENTIALS` environment variable
2. a JSON file in the well-known path `$HOME/.config/gcloud/application_default_credentials.json`
3. fetched from the GCE metadata server

If Prometheus is running within GCE, the service account associated with the
instance it is running on should have at least read-only permissions to the
compute resources. If running outside of GCE make sure to create an appropriate
service account and place the credential file in one of the expected locations.

### `<kubernetes_sd_config>`

Kubernetes SDの設定によって、[Kubernetes](https://kubernetes.io/)のREST APIからスクレイプ対象を取得でき、クラスタの状態と常に動機を保つ事が出来る。

監視対象を検出するために、以下の`role`の1つを設定出来る。

#### `node`

`node`ロールはクラスタノードにつき1つの監視対象を、KubeletのHTTPポートをデフォルトとしたアドレスと共に検出する。
最初の存在したアドレスKubernetesノードオブジェクト
The `node` role discovers one target per cluster node with the address defaulting
to the Kubelet's HTTP port.
The target address defaults to the first existing address of the Kubernetes
node object in the address type order of `NodeInternalIP`, `NodeExternalIP`,
`NodeLegacyHostIP`, and `NodeHostName`.

Available meta labels:

* `__meta_kubernetes_node_name`: The name of the node object.
* `__meta_kubernetes_node_label_<labelname>`: Each label from the node object.
* `__meta_kubernetes_node_labelpresent_<labelname>`: `true` for each label from the node object.
* `__meta_kubernetes_node_annotation_<annotationname>`: Each annotation from the node object.
* `__meta_kubernetes_node_annotationpresent_<annotationname>`: `true` for each annotation from the node object.
* `__meta_kubernetes_node_address_<address_type>`: The first address for each node address type, if it exists.

In addition, the `instance` label for the node will be set to the node name
as retrieved from the API server.

#### `service`

The `service` role discovers a target for each service port for each service.
This is generally useful for blackbox monitoring of a service.
The address will be set to the Kubernetes DNS name of the service and respective
service port.

Available meta labels:

* `__meta_kubernetes_namespace`: The namespace of the service object.
* `__meta_kubernetes_service_annotation_<annotationname>`: The annotation of the service object.
* `__meta_kubernetes_service_annotation_<annotationname>`: Each annotation from the service object.
* `__meta_kubernetes_service_annotationpresent_<annotationname>`: "true" for each annotation of the service object.
* `__meta_kubernetes_service_cluster_ip`: The cluster IP address of the service. (Does not apply to services of type ExternalName)
* `__meta_kubernetes_service_external_name`: The DNS name of the service. (Applies to services of type ExternalName)
* `__meta_kubernetes_service_label_<labelname>`: Each label from the service object.
* `__meta_kubernetes_service_labelpresent_<labelname>`: `true` for each label of the service object.
* `__meta_kubernetes_service_name`: The name of the service object.
* `__meta_kubernetes_service_port_name`: Name of the service port for the target.
* `__meta_kubernetes_service_port_number`: Number of the service port for the target.
* `__meta_kubernetes_service_port_protocol`: Protocol of the service port for the target.

#### `pod`

The `pod` role discovers all pods and exposes their containers as targets. For each declared
port of a container, a single target is generated. If a container has no specified ports,
a port-free target per container is created for manually adding a port via relabeling.

Available meta labels:

* `__meta_kubernetes_namespace`: The namespace of the pod object.
* `__meta_kubernetes_pod_name`: The name of the pod object.
* `__meta_kubernetes_pod_ip`: The pod IP of the pod object.
* `__meta_kubernetes_pod_label_<labelname>`: Each label from the pod object.
* `__meta_kubernetes_pod_labelpresent_<labelname>`: `true`for each label from the pod object.
* `__meta_kubernetes_pod_annotation_<annotationname>`: Each annotation from the pod object.
* `__meta_kubernetes_pod_annotationpresent_<annotationname>`: `true` for each annotation from the pod object.
* `__meta_kubernetes_pod_container_init`: `true` if the container is an [InitContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
* `__meta_kubernetes_pod_container_name`: Name of the container the target address points to.
* `__meta_kubernetes_pod_container_port_name`: Name of the container port.
* `__meta_kubernetes_pod_container_port_number`: Number of the container port.
* `__meta_kubernetes_pod_container_port_protocol`: Protocol of the container port.
* `__meta_kubernetes_pod_ready`: Set to `true` or `false` for the pod's ready state.
* `__meta_kubernetes_pod_phase`: Set to `Pending`, `Running`, `Succeeded`, `Failed` or `Unknown`
  in the [lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase).
* `__meta_kubernetes_pod_node_name`: The name of the node the pod is scheduled onto.
* `__meta_kubernetes_pod_host_ip`: The current host IP of the pod object.
* `__meta_kubernetes_pod_uid`: The UID of the pod object.
* `__meta_kubernetes_pod_controller_kind`: Object kind of the pod controller.
* `__meta_kubernetes_pod_controller_name`: Name of the pod controller.

#### `endpoints`

The `endpoints` role discovers targets from listed endpoints of a service. For each endpoint
address one target is discovered per port. If the endpoint is backed by a pod, all
additional container ports of the pod, not bound to an endpoint port, are discovered as targets as well.

Available meta labels:

* `__meta_kubernetes_namespace`: The namespace of the endpoints object.
* `__meta_kubernetes_endpoints_name`: The names of the endpoints object.
* For all targets discovered directly from the endpoints list (those not additionally inferred
  from underlying pods), the following labels are attached:
  * `__meta_kubernetes_endpoint_hostname`: Hostname of the endpoint.
  * `__meta_kubernetes_endpoint_node_name`: Name of the node hosting the endpoint.
  * `__meta_kubernetes_endpoint_ready`: Set to `true` or `false` for the endpoint's ready state.
  * `__meta_kubernetes_endpoint_port_name`: Name of the endpoint port.
  * `__meta_kubernetes_endpoint_port_protocol`: Protocol of the endpoint port.
  * `__meta_kubernetes_endpoint_address_target_kind`: Kind of the endpoint address target.
  * `__meta_kubernetes_endpoint_address_target_name`: Name of the endpoint address target.
* If the endpoints belong to a service, all labels of the `role: service` discovery are attached.
* For all targets backed by a pod, all labels of the `role: pod` discovery are attached.

#### `ingress`

The `ingress` role discovers a target for each path of each ingress.
This is generally useful for blackbox monitoring of an ingress.
The address will be set to the host specified in the ingress spec.

Available meta labels:

* `__meta_kubernetes_namespace`: The namespace of the ingress object.
* `__meta_kubernetes_ingress_name`: The name of the ingress object.
* `__meta_kubernetes_ingress_label_<labelname>`: Each label from the ingress object.
* `__meta_kubernetes_ingress_labelpresent_<labelname>`: `true` for each label from the ingress object.
* `__meta_kubernetes_ingress_annotation_<annotationname>`: Each annotation from the ingress object.
* `__meta_kubernetes_ingress_annotationpresent_<annotationname>`: `true` for each annotation from the ingress object.
* `__meta_kubernetes_ingress_scheme`: Protocol scheme of ingress, `https` if TLS
  config is set. Defaults to `http`.
* `__meta_kubernetes_ingress_path`: Path from ingress spec. Defaults to `/`.

See below for the configuration options for Kubernetes discovery:

```yaml
# The information to access the Kubernetes API.

# The API server addresses. If left empty, Prometheus is assumed to run inside
# of the cluster and will discover API servers automatically and use the pod's
# CA certificate and bearer token file at /var/run/secrets/kubernetes.io/serviceaccount/.
[ api_server: <host> ]

# The Kubernetes role of entities that should be discovered.
role: <role>

# Optional authentication information used to authenticate to the API server.
# Note that `basic_auth`, `bearer_token` and `bearer_token_file` options are
# mutually exclusive.
# password and password_file are mutually exclusive.

# Optional HTTP basic authentication information.
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# Optional bearer token authentication information.
[ bearer_token: <secret> ]

# Optional bearer token file authentication information.
[ bearer_token_file: <filename> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# TLS configuration.
tls_config:
  [ <tls_config> ]

# Optional namespace discovery. If omitted, all namespaces are used.
namespaces:
  names:
    [ - <string> ]
```

Where `<role>` must be `endpoints`, `service`, `pod`, `node`, or
`ingress`.

See [this example Prometheus configuration file](/documentation/examples/prometheus-kubernetes.yml)
for a detailed example of configuring Prometheus for Kubernetes.

You may wish to check out the 3rd party [Prometheus Operator](https://github.com/coreos/prometheus-operator),
which automates the Prometheus setup on top of Kubernetes.

### `<marathon_sd_config>`

Marathon SD configurations allow retrieving scrape targets using the
[Marathon](https://mesosphere.github.io/marathon/) REST API. Prometheus
will periodically check the REST endpoint for currently running tasks and
create a target group for every app that has at least one healthy task.

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_marathon_app`: the name of the app (with slashes replaced by dashes)
* `__meta_marathon_image`: the name of the Docker image used (if available)
* `__meta_marathon_task`: the ID of the Mesos task
* `__meta_marathon_app_label_<labelname>`: any Marathon labels attached to the app
* `__meta_marathon_port_definition_label_<labelname>`: the port definition labels
* `__meta_marathon_port_mapping_label_<labelname>`: the port mapping labels
* `__meta_marathon_port_index`: the port index number (e.g. `1` for `PORT1`)

See below for the configuration options for Marathon discovery:

```yaml
# List of URLs to be used to contact Marathon servers.
# You need to provide at least one server URL.
servers:
  - <string>

# Polling interval
[ refresh_interval: <duration> | default = 30s ]

# Optional authentication information for token-based authentication
# https://docs.mesosphere.com/1.11/security/ent/iam-api/#passing-an-authentication-token
# It is mutually exclusive with `auth_token_file` and other authentication mechanisms.
[ auth_token: <secret> ]

# Optional authentication information for token-based authentication
# https://docs.mesosphere.com/1.11/security/ent/iam-api/#passing-an-authentication-token
# It is mutually exclusive with `auth_token` and other authentication mechanisms.
[ auth_token_file: <filename> ]

# Sets the `Authorization` header on every request with the
# configured username and password.
# This is mutually exclusive with other authentication mechanisms.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file` and other authentication mechanisms.
# NOTE: The current version of DC/OS marathon (v1.11.0) does not support standard Bearer token authentication. Use `auth_token` instead.
[ bearer_token: <string> ]

# Sets the `Authorization` header on every request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token` and other authentication mechanisms.
# NOTE: The current version of DC/OS marathon (v1.11.0) does not support standard Bearer token authentication. Use `auth_token_file` instead.
[ bearer_token_file: /path/to/bearer/token/file ]

# TLS configuration for connecting to marathon servers
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
```

By default every app listed in Marathon will be scraped by Prometheus. If not all
of your services provide Prometheus metrics, you can use a Marathon label and
Prometheus relabeling to control which instances will actually be scraped.
See [the Prometheus marathon-sd configuration file](/documentation/examples/prometheus-marathon.yml)
for a practical example on how to set up your Marathon app and your Prometheus
configuration.

By default, all apps will show up as a single job in Prometheus (the one specified
in the configuration file), which can also be changed using relabeling.

### `<nerve_sd_config>`

Nerve SD configurations allow retrieving scrape targets from [AirBnB's Nerve]
(https://github.com/airbnb/nerve) which are stored in
[Zookeeper](https://zookeeper.apache.org/).

The following meta labels are available on targets during [relabeling](#relabel_config):

* `__meta_nerve_path`: the full path to the endpoint node in Zookeeper
* `__meta_nerve_endpoint_host`: the host of the endpoint
* `__meta_nerve_endpoint_port`: the port of the endpoint
* `__meta_nerve_endpoint_name`: the name of the endpoint

```yaml
# The Zookeeper servers.
servers:
  - <host>
# Paths can point to a single service, or the root of a tree of services.
paths:
  - <string>
[ timeout: <duration> | default = 10s ]
```

### `<serverset_sd_config>`

Serverset SD configurations allow retrieving scrape targets from [Serversets]
(https://github.com/twitter/finagle/tree/master/finagle-serversets) which are
stored in [Zookeeper](https://zookeeper.apache.org/). Serversets are commonly
used by [Finagle](https://twitter.github.io/finagle/) and
[Aurora](https://aurora.apache.org/).

The following meta labels are available on targets during relabeling:

* `__meta_serverset_path`: the full path to the serverset member node in Zookeeper
* `__meta_serverset_endpoint_host`: the host of the default endpoint
* `__meta_serverset_endpoint_port`: the port of the default endpoint
* `__meta_serverset_endpoint_host_<endpoint>`: the host of the given endpoint
* `__meta_serverset_endpoint_port_<endpoint>`: the port of the given endpoint
* `__meta_serverset_shard`: the shard number of the member
* `__meta_serverset_status`: the status of the member

```yaml
# The Zookeeper servers.
servers:
  - <host>
# Paths can point to a single serverset, or the root of a tree of serversets.
paths:
  - <string>
[ timeout: <duration> | default = 10s ]
```

Serverset data must be in the JSON format, the Thrift format is not currently supported.

### `<triton_sd_config>`

[Triton](https://github.com/joyent/triton) SD configurations allow retrieving
scrape targets from [Container Monitor](https://github.com/joyent/rfd/blob/master/rfd/0027/README.md)
discovery endpoints.

The following meta labels are available on targets during relabeling:

* `__meta_triton_groups`: the list of groups belonging to the target joined by a comma separator
* `__meta_triton_machine_alias`: the alias of the target container
* `__meta_triton_machine_brand`: the brand of the target container
* `__meta_triton_machine_id`: the UUID of the target container
* `__meta_triton_machine_image`: the target containers image type
* `__meta_triton_server_id`: the server UUID for the target container

```yaml
# The information to access the Triton discovery API.

# The account to use for discovering new target containers.
account: <string>

# The DNS suffix which should be applied to target containers.
dns_suffix: <string>

# The Triton discovery endpoint (e.g. 'cmon.us-east-3b.triton.zone'). This is
# often the same value as dns_suffix.
endpoint: <string>

# A list of groups for which targets are retrieved. If omitted, all containers
# available to the requesting account are scraped.
groups:
  [ - <string> ... ]

# The port to use for discovery and metric scraping.
[ port: <int> | default = 9163 ]

# The interval which should be used for refreshing target containers.
[ refresh_interval: <duration> | default = 60s ]

# The Triton discovery API version.
[ version: <int> | default = 1 ]

# TLS configuration.
tls_config:
  [ <tls_config> ]
```

### `<static_config>`

`static_config`によって、監視対象のリストとそれらに共通のラベル集合を指定できる。
スクレイプの設定の中で静的な監視対象を指定するもっとも簡潔な方法である。

```yaml
# 静的な設定で指定される監視対象
targets:
  [ - '<host>' ]

# targetsからスクレイプされた全てのメトリクスに割り当てられるラベル
labels:
  [ <labelname>: <labelvalue> ... ]
```

### `<relabel_config>`

リラベルは、監視対象のラベル集合を、スクレイプされる前に、動的に書き換えるための強力な手段である。
スクレイプの設定毎に複数のリラベルのステップを設定できる。
それらのステップは、各監視対象のラベル集合に、設定ファイルに出てくる順に適用される。

まずはじめに、監視対象毎に設定されたラベルを除いて、対応するscrapeの設定の`job_name`の値に`job`ラベルがセットされる。
`__address__`ラベルが監視対象のアドレス`<host>:<port>`にセットされる。リラベルの後で、`instance`ラベルがセットされていなければ、デフォルトで`__address__`の値にセットされる。`__scheme__`ラベルと`__metrics_path__`ラベルはそれぞれ、監視対象のスキームとメトリクスのパスにセットされる。`__param_<name>`ラベルは、`<name>`という名前で渡された最初のURLパラメーターの値にセットされる。

リラベルの過程で、`__meta_`でプリフィックスされたラベルが利用可能な場合がある。そういうラベルは、その監視対象を提供しているサービスディスカバリーの仕組みによってセットされる。それらのラベルはサービスディスカバリーの仕組みによって異なる。

`__`で始まるラベルは、リラベル完了後にラベル集合から取り除かれる。

もしリラベルの過程で（後続の理ラベルの入力として）一時的にラベルの値を保存する必要があるなら、`__tmp`というプリフィックスを使うこと。このプリフィックスはPrometheus自体には使われないことが保証されている。

```yaml
# 存在しているラベルから値を選択する。それらの内容は、設定されたseparatorで連結され、置
# 換、保存、削除のために、設定された正規表現にマッチングされる。
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
[ separator: <string> | default = ; ]

# replaceで、結果の値を書き込むためのラベル
# replaceする場合は、必須項目。正規表現のキャプチャされたグループ利用可能
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <uint64> ]

# 置換する値。正規表現がマッチした場合、この値で正規表現置換される。
# 正規表現でキャプチャされたグループが利用可能
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```

`<regex>`は、[RE2](https://github.com/google/re2/wiki/Syntax)で書かれた正しい正規表現である。
これは、`replace`、`keep`、`drop`、`labelmap`、`labeldrop`、`labelkeep`のアクションのために必須である。
この正規表現は、両端がアンカーされている。アンカーをなくすためには、`.*<regex>.*`を使うこと。

`<relabel_action>`は、リラベルのアクションを決める。

* `replace`: 連結された`source_labels`に対して`regex`をマッチをし、`target_label`の値を`replacement`にセットする。`replacement`のグループ参照（`${1}`、`${2}`など）はそれらの値で置換される。
* `keep`: 連結された`source_labels`に`regex`がマッチしないターゲットをdropする
* `drop`: 連結された`source_labels`に`regex`がマッチするターゲットを削除する
* `hashmod`: `target_label`の値を、連結された`source_labels`のハッシュを`modulus`でmodした値にセットする
* `labelmap`: 全てのラベル名に対して`regex`をマッチをし、マッチしたラベルの値を`replacement`で指定されたラベルにコピーする。`replacement`のグループ参照（`${1}`、`${2}`など）はそれらの値で置換される。
* `labeldrop`: 全てのラベル名に対して`regex`をマッチをし、マッチしたラベルはラベル集合から削除される
* `labelkeep`: 全てのラベル名に対して`regex`をマッチをし、マッチしなかったラベルはラベル集合から削除される

`labeldrop`と`labelkeep`を使うときには、ラベルが削除されてもメトリクスが確実にユニークにラベルされているように注意する必要がある

### `<metric_relabel_configs>`

メトリックのリラベルは、取り込まれる前の最後のステップとしてサンプルに適用される。
ターゲットのリラベルと同じ設定フォーマットとアクションを持つ。
メトリックのリラベルは、`up`などの生成された時系列には自動的には適用されない。

この機能の使い道の1つは、取り込むコストが高い時系列のブラックリストで弾くことである。

### `<alert_relabel_configs>`

アラートのリラベルは、アラートがAlertmanagerに送信される前に適用される。
ターゲットのリラベルと同じ設定フォーマットとアクションを持つ。
アラートのリラベルは、外部ラベルのあとで適用される。

この機能の使い道の1つは、異なる外部ラベルを持つHAペアのPrometheusサーバーが全く同じアラートを送ることを確実にすることである。

### `<alertmanager_config>`

`alertmanager_config`は、Prometheusサーバーがアラートを送るAlertmanagerのインスタンスを指定する。
Alertmanagerとどのように通信するかを設定するパラメーターも提供する。

Alertmanagerは、`static_configs`によって静的に設定することも、サポートされているサービスディスカバリーで動的に検出することもできる。

`relabel_configs`によって、検出された中からAlertmanagerを選択したり、`__alerts_path__`ラベルで出力されるAPIのパスに対して高度な変更をすることができる。

```yaml
# Per-target Alertmanager timeout when pushing alerts.
[ timeout: <duration> | default = 10s ]

# The api version of Alertmanager.
[ api_version: <version> | default = v1 ]

# Prefix for the HTTP path alerts are pushed to.
[ path_prefix: <path> | default = / ]

# Configures the protocol scheme used for requests.
[ scheme: <scheme> | default = http ]

# 設定されたusernameとpasswordでscrapeのリクエストの`Authorization`ヘッダを毎回セットする。
# passwordとpassword_fileは相互排他的である。
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# 設定された署名なしトークン (Bearer Token)でリクエストの`Authorization`ヘッダを
# 毎回セットする。`bearer_token_file`と相互排他的である。
[ bearer_token: <string> ]

# 設定ファイルから読み込んだ署名なしトークン (Bearer Token)でリクエストの
# `Authorization`ヘッダを毎回セットする。`bearer_token`と相互排他的である。
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the scrape request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# List of Azure service discovery configurations.
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# List of Consul service discovery configurations.
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# List of DNS service discovery configurations.
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# List of EC2 service discovery configurations.
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# List of file service discovery configurations.
file_sd_configs:
  [ - <file_sd_config> ... ]

# List of GCE service discovery configurations.
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# List of Kubernetes service discovery configurations.
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# List of Marathon service discovery configurations.
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# List of AirBnB's Nerve service discovery configurations.
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# List of Zookeeper Serverset service discovery configurations.
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# List of Triton service discovery configurations.
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# List of labeled statically configured Alertmanagers.
static_configs:
  [ - <static_config> ... ]

# List of Alertmanager relabel configurations.
relabel_configs:
  [ - <relabel_config> ... ]
```

### `<remote_write>`

`write_relabel_configs`は、リモートエンドポイントに送信される前にサンプルに適用されるリラベルである。
writeリラベルは、外部のラベルの後に適用される。
これは、どのサンプルが送信されるかを制限するために利用できるだろう。

この機能をどのように使うかの[デモ](/documentation/examples/remote_storage)がある。

```yaml
# サンプルを送信する先のエンドポイントURL
url: <string>

# remote writeエンドポイントへのリクエストのタイムアウト
[ remote_timeout: <duration> | default = 30s ]

# remote writeのリラベル設定のリスト
write_relabel_configs:
  [ - <relabel_config> ... ]

# 設定されたusernameとpasswordでremote writeのリクエストの`Authorization`ヘッダを毎回セットする。
# passwordとpassword_fileは相互排他的である。
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# 設定された署名なしトークン (Bearer Token)でremote writeのリクエストの`Authorization`ヘッダを
# 毎回セットする。`bearer_token_file`と相互排他的である。
[ bearer_token: <string> ]

# 設定ファイルから読み込んだ署名なしトークン (Bearer Token)でremote writeのリクエストの
# `Authorization`ヘッダを毎回セットする。`bearer_token`と相互排他的である。
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the remote write request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# Configures the queue used to write to remote storage.
queue_config:
  # Number of samples to buffer per shard before we block reading of more samples from the WAL.
  [ capacity: <int> | default = 10 ]
  # Maximum number of shards, i.e. amount of concurrency.
  [ max_shards: <int> | default = 1000 ]
  # Minimum number of shards, i.e. amount of concurrency.
  [ min_shards: <int> | default = 1 ]
  # Maximum number of samples per send.
  [ max_samples_per_send: <int> | default = 100]
  # Maximum time a sample will wait in buffer.
  [ batch_send_deadline: <duration> | default = 5s ]
  # Initial retry delay. Gets doubled for every retry.
  [ min_backoff: <duration> | default = 30ms ]
  # Maximum retry delay.
  [ max_backoff: <duration> | default = 100ms ]

```

この機能との[連携のリスト](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)がある。

### `<remote_read>`

```yaml
# The URL of the endpoint to query from.
url: <string>

# remote readエンドポイントにクエリするためのセレクター
# に含まれていなければいけない等値マッチャーのリスト（オプショナル）
required_matchers:
  [ <labelname>: <labelvalue> ... ]

# remote readエンドポイントへのリクエストのタイムアウト
[ remote_timeout: <duration> | default = 1m ]

# ローカルストレージが完全なデータを持っているはずの時間の範囲に対して読み込みをするべきか
[ read_recent: <boolean> | default = false ]

# 設定されたusernameとpasswordでremote readのリクエストの`Authorization`ヘッダを毎回セットする。
# passwordとpassword_fileは相互排他的である。
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# 設定された署名なしトークン (Bearer Token)でremote readのリクエストの`Authorization`ヘッダを
# 毎回セットする。`bearer_token_file`と相互排他的である。
[ bearer_token: <string> ]

# 設定ファイルから読み込んだ署名なしトークン (Bearer Token)でremote readのリクエストの
# `Authorization`ヘッダを毎回セットする。`bearer_token`と相互排他的である。
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the remote read request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
```

この機能との[連携のリスト](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)がある。
