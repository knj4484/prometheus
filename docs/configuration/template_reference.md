---
title: テンプレートのリファレンス
sort_rank: 5
---

# テンプレートのリファレンス

Prometheusは、コンソールページと同様に、アラートのアノテーションとラベルでテンプレートをサポートしている。
テンプレートで、ローカルのデータベースに対してクエリを実行したり、繰り返し処理、条件分岐、フォーマットなどをすることが出来る。
Prometheusのテンプレート言語は[Goのテンプレートシステム](https://golang.org/pkg/text/template/)に基づいている。

## データ構造

時系列を扱うための最も重要なデータ構造は、以下のように定義されているサンプルである。

```go
type sample struct {
        Labels map[string]string
        Value  float64
}
```

サンプルのメトリック名は、マップ`Labels`の特別なラベル`__name__`で表現されている。

`[]sample`は、サンプルのリストを意味する。

Goの`interface{}`は、Cのvoidポインターと似ている。

## 関数

Goテンプレートが提供している[デフォルト関数](https://golang.org/pkg/text/template/#hdr-Functions)に加えて、Prometheusは、テンプレートの中でクエリー結果の処理を簡単にするための関数を提供している。

関数がパイプラインで使われている場合、パイプラインの値は最後の引数として渡される。

### クエリ

|  名前         | 引数          | 戻り値   | 備考     |
| ------------- | ------------- | -------- | -------- |
| query         | query string  | []sample | データベースを検索する。rangevectorの返却はサポートしていない  |
| first         | []sample      | sample   | `index a 0`と同じ |
| label         | label, sample | string   | `index sample.Labels label`と同じ |
| value         | sample        | float64  | `sample.Value` |
| sortByLabel   | label, []samples | []sample | サンプルを指定されたラベルでソートする。安定なソートである。 |

`first`、`label`、`value`は、クエリーの結果をパイプラインで簡単に使えるようにするためにある。

### 数値

|  名前         | 引数          | 戻り値   | 備考     |
| ------------- | --------------| --------| --------- |
| humanize      | number        | string  | [SI接頭辞](https://ja.wikipedia.org/wiki/SI接頭辞)を用いて、読みやすい形式に変換する |
| humanize1024  | number        | string  | `humanize`に似ているが、基数として1000ではなく1024を用いる |
| humanizeDuration | number     | string  | 秒単位の時間を読みやすい形式に変換する |
| humanizeTimestamp | number    | string  | 秒単位のUnixタイムスタンプを読みやすい形式に変換する |

humanize関数は、人間が利用するために適切な出力を生成するためにあり、Prometheusのバージョン間で同じ結果になることは保証されていない。

### 文字列

|  名前         | 引数          | 戻り値   | 備考     |
| ------------- | ------------- | ------- | ----------- |
| title         | string        | string  | [strings.Title](https://golang.org/pkg/strings/#Title), 各単語の最初の文字を大文字にする |
| toUpper       | string        | string  | [strings.ToUpper](https://golang.org/pkg/strings/#ToUpper), 全ての文字を大文字にする |
| toLower       | string        | string  | [strings.ToLower](https://golang.org/pkg/strings/#ToLower), 全ての文字を小文字にする |
| match         | pattern, text | boolean | [regexp.MatchString](https://golang.org/pkg/regexp/#MatchString) アンカーなしの正規表現マッチをテストする |
| reReplaceAll  | pattern, replacement, text | string | [Regexp.ReplaceAllString](https://golang.org/pkg/regexp/#Regexp.ReplaceAllString) 正規表現置換。アンカーなし |
| graphLink  | expr | string | 引数の式に対する[expressionブラウザ](https://prometheus.io/docs/visualization/browser/)のグラフビューへのパスを返す |
| tableLink  | expr | string | 引数の式に対する[expressionブラウザ](https://prometheus.io/docs/visualization/browser/)の表（Console）ビューへのパスを返す |

### その他

|  名前         | 引数          | 戻り値   | 備考     |
| ------------- | ------------- | ------- | ----------- |
| args          | []interface{} | map[string]interface{} | オブジェクトのリストを、arg0、arg1などをキーに持つマップに変換する。これはテンプレートに複数の引数を渡すことが意図されている |
| tmpl          | string, []interface{} | なし  | ビルトインの`template`と似ているが、テンプレート名にリテラルでないものが許される。結果が安全であることが仮定されており、自動的にエスケープされることはない。コンソールでのみ利用可能 |
| safeHtml      | string        | string  | 自動的なエスケープを必要としないものとしてHTMLをマークする |

## Template type differences

テンプレートの種類ごとに、テンプレートをパラメーター化するために利用可能な異なる情報を提供し、また、その他にも少しずつ違いがある。

### アラートのフィールドのテンプレート

`.Value`と`.Labels`には、アラートの値とラベルが入っている。
利便性のために、これらは変数`$value`と`$labels`にも割り当てられている。

### コンソールのテンプレート

コンソールは、`/consoles/`で見ることができ、フラグ`-web.console.templates`で指定されたディレクトリが元になっている。

コンソールテンプレートは、自動エスケープを行う[html/template](https://golang.org/pkg/html/template/)で展開される。
自動エスケープを回避するには、関数`safe*`を利用する。

URLパラメーターは、`.Params`でマップとして利用可能である。
同じ名前の複数のURLパラメーターを利用するために、`.RawParams`が各パラメーターの値のリストのマップになっている。
URLパス（プリフィックス`/consoles/`は除く）は、`.Path`で利用可能である。

コンソールは、フラグ`-web.console.libraries`で指定されたディレクトリ内の`*.lib`ファイルにある`{{define "templateName"}}...{{end}}`で定義されたテンプレート全てを利用可能である。
この名前空間は共有なので、他のユーザーとの衝突を回避するように気をつけること。
`prom`、`_prom`、`__`で始まるテンプレート名は、上で一覧にされている関数と同様に、Prometheusによる利用のために予約されている。
