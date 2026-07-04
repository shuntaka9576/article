---
title: "簡単なツールを書くのにgoogle/zxが便利だった話"
type: "tech"
description: "google/zxが便利だったので、Tipsとか"
publish: true
tags:
  - "tech/typescript"
  - "tech/github"
  - "tech/cli"
---

# はじめに

シェルスクリプトは、複数の外部コマンドを実行したり、コマンドの標準出力を扱うといった、ライトで単純なユースケースで便利です。CI/CD, ログ解析, 業務効率化ツール周りで使うことが多い印象です。
一方でJSONをパースしたり、連想配列を取り回す必要が出てくるとシェルスクリプトだと厳しいので、私の場合はPerlを使っていました。(※)

単なるコマンド実行の羅列であれば、シェルスクリプトで良いと思います。ただ、外部コマンドを使いつつ、複雑なリスト操作もしたいけど、、、PythonやGoでSDKを使ってゴリゴリ実装するのはちょっと...みたいな需要があるように思います。

上記のケースで[google/zx](https://github.com/google/zx)を利用し非常に便利だったため、記事にしました。

※ 良識のある大人なので、Perlに馴染みのないメンバーが多いプロジェクトでPerlを突っ込んだことはありません(ログ解析のワンライナーくらい 脱法Perl)

# 経緯
少し前に話題になった[シェルスクリプトを書くのをやめる](https://blog.8-p.info/ja/2021/09/15/bash/)がとても良い記事で、分かりみが深みでし
た。Perlに関しては、記事にはないものの筆者のツイートで言及がありました。

[前述の記事の筆者のツイート](https://twitter.com/kzys/status/1438122674400595968)
> 我々中年プログラマ諸氏はPerl書けるけど封印しているじゃないですか。

記事内で、google/zxに言及がありました
> Go や Rust も良いけれど、ビルドの過程を理解できなくても、実行しているものを直接読めるというスクリプト言語の利点は捨てがたい。zx や Oil は気になるけれど、まだ試していない。

[zx - より良いスクリプトを書くためのツール](https://www.infoq.com/jp/news/2021/07/zx-javascript-cli-scripts/)で、`google/zx`は知っていたが、丁度少し重めのPerlのスクリプトを書いていたこともあり、試してみることにしました。


# Perlの良いところ
zxの記事ですが、少しお付き合いください..

実際のユースケースで考えてみます。GitHubリポジトリのPR一覧で、以下の条件を全て満たすツールを作る場合を想定します。
* PRタイトルに特定の文字列を含まない
* `{PR番号: "PRタイトル"...}`のようなオブジェクトを作りたい

GoでGitHub API v4を`net/http`やGitHubのSDKを使って、実装しちゃうぞ〜とかやると、慣れていないと結構面倒だったりします。GitHub CLIとPerlで書くとこんな感じになります。

```bash:GitHub CLIを使ったPR一覧の取得
$ gh pr list --state merged
#50  ライブラリのバージョン固定とアップデート                                                        feature/update-library
#49  特定の文字列                                                                                     develop
#48  カテゴリヘッダーを追加                                                                          feature/add-category
#45  Chpaterページのレイアウト調整                                                                   feature/adjustment-doc-page
#44  ドキュメント機能の仮実装                                                                        feature/support-doc
```

```perl: 特定のタイトルのPRを一覧を取得するPerlスクリプト
# リポジトリのPR一覧を取得し、1列目と2列目にawkでフィルタする
my $pr_list=`gh pr list --repo $repoName --limit $GET_PR_LIST_NUMBER --state merged | awk -F '\t' '{print \$1,\$2}'`;
my %pr_tilte_without_str_hash = (); # ハッシュの宣言

foreach my $pr_title (split(/\n/, $pr_list)) { # 出力結果を\nで区切る
  if ($pr_title =~ /^(\d+?)\s*?特定の文字列.*$/ ){
  } elsif ($pr_title =~ /^(\d+?)\s+?(.*)$/ ) { # PR番号を$1にタイトルを$2に格納する
    $pr_tilte_without_str_hash{$1}=$2;
  }
}
```
GitHub CLIからJSON形式で標準出力に書き出し、Perlでパースすればもっとシンプルになるかもしれません。(私にPerl力がなく、、)

Perlは以下の点で、メリットがあります
* バッククォートで\`コマンド\` を実行でき、標準出力を受け取れる
-> Pythonだったらsubprocess。Goだったらos/execのようなプロセス操作系のライブラリを使わずに済む
* 正規表現(PCRE)が強力で、=~で出力結果を抽出できる
* シェルスクリプトより配列やハッシュが取り回しやすい(とはいえ個人的にはクセが強い..

# [google/zx](https://github.com/google/zx)

> A tool for writing better scripts

とあるように、TS/JSをシェルスクリプトみたいに使えるくらい思っておけば十分だと思います。

# google/zxを使う

## 導入
```bash
npm i -g zx
```

基礎的な使い方は、[公式](https://github.com/google/zx#documentation)を参照

## 実装

先ほどPerlで実装したスクリプトをzxで書くと以下のようになります。

> * PRタイトルに特定の文字列を含まない
> * `{PR番号: "PRタイトル"...}`のようなオブジェクトを作りたい

```js
const execGhPrListReuslt = await $`gh pr list \
  --repo ${repoName} \
  --limit 1000 \
  --json 'number,title,mergedAt,body,state' \
  --state merged`
const prJsonList = JSON.parse(execGhPrListReuslt.stdout)

const pr_tilte_without_str_hash = {};
const strRegExp = new RegExp(`特定の文字列`)

prJsonList.filter(pr => strRegExp.test(pr.title) === false)
  .map((pr) => {
    pr_tilte_without_str_hash[pr.number] = pr.title
  })
```

## 便利な機能
### リモート実行 
HTTPでソースコードが取得できれば、直接実行可能です。`--quiet`オプションは、コマンドをechoするのでデバックのとき以外は必要です。

```bash
$ zx --quiet https://raw.githubusercontent.com/zxshdev/zxsh.dev/master/scripts/hello.mjs
Hello zxsh.dev!!!
```

# TypeScriptを使う

:::message
TypeScriptで書く場合、zxをimportして利用することになるため、普通にTSを書いているのと同じです。
:::

## はじめに

zxには、TypeScriptを直接実行する機能あります。ただHTTP経由でリモート実行しようとすると失敗します。

```bash
$ zx https://raw.githubusercontent.com/zxshdev/zx-ts-template/master/index.ts
$ fetch https://raw.githubusercontent.com/zxshdev/zx-ts-template/master/index.ts
error TS6053: File '/var/folders/k_/ztkld5cj5z959s72w9r630200000gq/T/index.ts' not found.
  The file is in the program because:
    Root file specified for compilation
```

上記の回避策として、webpackを利用して、jsにトランスパイル/バンドルしてホストしたところ動作しました。
設定ファイル(webpack.config.jsやtsconfig.js)の内容は、[zx-ts-template](https://github.com/zxshdev/zx-ts-template)リポジトリを作成しましたので、参考になれば。

## 使い方

[zx-ts-template](https://github.com/zxshdev/zx-ts-template)は、下記のtsファイルをjsにトランスパイル/バンドルしてzxでリモート実行するのがゴールです。サンプルでは時刻操作ライブラリの`luxon`をバンドルします。

```ts:index.ts
import {$} from 'zx'
import {DateTime} from 'luxon'

const main = async () => {
  const result = await $`ls`
  console.log(`ls result: ${result.stdout}`)

  const now = DateTime.now().toMillis()
  console.log(`now: ${now}`)
}

main().then(() => console.log('Success!'))
```

```bash
git clone https://github.com/zxshdev/zx-ts-template.git
# 外部ライブラリの依存解決
yarn install
# dist/index.js`にバンドルされたファイルが作成されます。
yarn build
```

tsでもバンドル後index.jsでもzxで動作します

```bash: ローカルで実行する場合
zx index.ts
zx dist/index.js
# 当たり前ですが、zxはimportして使う場合、単なるライブラリなのでnodeで動作します
node dist/index.js
```


ビルドしたjsを、GitHubやサーバーに配置すれば、下記のようにリモート実行可能です

```bash
$ zx https://raw.githubusercontent.com/zxshdev/zxsh.dev/master/scripts/ts-template.js
$ fetch https://raw.githubusercontent.com/zxshdev/zxsh.dev/master/scripts/ts-template.js
$ ls
README.md
dist
index.ts
node_modules
package.json
tsconfig.json
webpack.config.js
yarn.lock
ls result: README.md
dist
index.ts
node_modules
package.json
tsconfig.json
webpack.config.js
yarn.lock

now: 1633826078520
Success!
```

# 最後に
個人的に以下の点でメリットを感じ、積極的にgoogle/zxを使っていこうと思いました！

* シンプルな外部コマンド実行シンタックス
* JSの強力な機能が使える
  * 強力なリスト操作関数
  * JSONをそのままオブジェクトとして扱える
  * 複数のコマンドをPromise.allで非同期実行可能
* 外部ライブラリの依存解決なしに、HTTP(S)経由でリモート実行可能(多分)

特にGitHub CLIのようにJSON形式で標準出力に書き出せる場合、そのまま取り回せるのが最高の体験ですね！

誤りがありましたら、[こちら](https://github.com/hozi-dev/article)にissueとして起票して頂けると助かります。

