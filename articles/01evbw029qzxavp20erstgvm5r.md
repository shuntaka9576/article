---
title: "npmモジュールをモノレポで管理し、煩雑なリリース業務を効率化する"
type: "tech"
description: "npmモジュールを効率的に管理するツールやTipsを紹介します"
publish: true
tags:
  - "tech/typescript"
  - "tech/npm"
  - "tech/モノレポ"
---

# はじめに
ある程度サービス開発を続けているとリポジトリを跨いだ処理の共通化、ライブラリを作成する必要がある。Node.js/TypeScriptでは、npmを利用することでバージョン毎にビルド済資材を管理でき、非常に便利。

一方で1~3人のチームでライブラリが3~4つあった場合にそれぞれリポジトリを分けて管理すると、リリースタスクが非常に煩雑。例えば..
* ほぼ同じようなCI/CDパイプラインを各リポジトリで実装する必要あり
* 開発ツール(eslint,husky,pritter,etc..)のアップデートを各リポジトリ毎に行う必要あり

本記事では1つリポジトリで、なるべく人の手を使わずに複数のnpmモジュールのリリース管理をする方法について書く。以下注意点。
* [lernaでのmonorepoにおけるリリースフロー(Fixed/Independent)](https://efcl.info/2019/01/26/monorepo-release-flow/)の記事が良くまとまっており、本記事と合わせて読むことを推奨
* 完成系の[サンプルリポジトリ](https://github.com/sugoi-site/packages)に用意しおり、詳細な設定が気になる場合は参照して欲しい

# 実現したいこと
* リリース時のセマンティックバージョン推論
* 更新があるモジュールの同時リリース
* eslintやprettierといった開発ツールの共通化

# 利用するツールや概念
## lerna
[リポジトリ](https://github.com/lerna/lerna)。複数のパッケージでJavaScriptプロジェクトを管理するためのツール。Babel、React、Angular、 Ember、Meteor、Jestといったツールで利用されている。

## Yarn Workspaces
下記の記事を参考にした。ここは時間のあるとき再考したい。
* [LernaとYarn WorkspacesでMonorepo管理](https://blog.cybozu.io/entry/2020/04/21/080000)
* [Yarn workspaces から Lerna に移行した](https://qiita.com/acro5piano/items/c2a8dd3a75c99b704548)

lernaのみで管理することも可能だが下記の点でYarn Workspacesと併用することとした
* yarnによるコマンドの巻き上げで影響がでるケースが、現状の要件上なさそうだった
* 併用しない場合、`yarn add`によるパッケージ追加が出来なくなる。`lerna`コマンド経由でのパッケージインストールは少し癖があると感じた

パッケージ共有でインストール場合はコマンドは下記
```bash
lerna add eslint
```
任意のパッケージのみで利用したい場合は、`scope`を設定する必要がある
```bash
lerna add lodash --scope lib-a
```

併用すれば、各パッケージルートで`yarn add`出来るため、慣れているやり方でパッケージ追加/削除が可能となる。

## Conventional Commits
[Conventional Commits](https://github.com/conventional-changelog/commitlint/tree/master/@commitlint/config-conventional)
> 人間と機械が読みやすく、意味のあるコミットメッセージにするための仕様

本仕様に従うことで、`lerna`で[セマンティックバージョニング](https://semver.org/lang/ja/)ベースでのリリースバージョン推論が可能となる。


### Format
フォーマットは下記。optionalは任意項目。
><type>[optional scope]: <description>
>
>[optional body]
>
>[optional footer(s)]

各項目で名付け規則が存在する。コミットする際には、こちらの[ルール](https://github.com/conventional-changelog/commitlint/tree/master/@commitlint/config-conventional#rules)を参考にしたい。

### バージョン推論
`Conventional Commits`を通して、下記ようにバージョン推論が行われる

* `MAJOR`バージョンがインクリメントされるのは、bodyまたはfooterに`BREAKING CHANGE:`がある場合
* `MINOR` `PATCH`バージョンがインクリメントされるのは、下記の表参考

| type名 | 説明 | 補足 |
| --- | --- | --- |
| build | ビルド設定 |
| ci | CI設定 |
| chore | 雑事(カテゴライズする必要ないようなもの) |
| docs | ドキュメント |
| feat | 新機能 | `MINOR`バージョンに該当 |
| fix | バグフィックス | `PATCH`バージョンに該当 |
| perf | パフォーマンス |
| refactor | リファクタリング |
| revert | コミット取り消し(git revert) |
| style | コードスタイル修正 |
| test | テスト |

### 例

```bash
# ドキュメントを更新した場合
docs(readme): 起動方法を変更
# 新機能を追加した場合(scopeは無理してつける必要なし!)
feat: 起動方法を変更
# renovateが自動アップデートした場合
chore(deps): update dependency @types/jest to v26.0.20
```

# ツールの設定
## 完成系
詳細設定は[リポジトリ](https://github.com/sugoi-site/packages)を参考にして欲しい

## lerna
lerna.json
```json
{
  "npmClient": "yarn", // npmクライアントにyarnを指定
  "useWorkspaces": true, // Yarn Workspacesを有効化
  "packages": [
    "packages/*"
  ],
  "version": "independent" // ①
  "command": {
    "version": {
      "message": "chore(release): publish", // ②
      "conventionalCommits": true // conventionalCommitsを使ったリリーズバージョン推論の有効化
    }
  },
}
```
### ①について
lernaには、下記のモードが存在する
* [Fixed/Locked mode](https://github.com/lerna/lerna#fixedlocked-mode-default)
* [Independent mode](https://github.com/lerna/lerna#independent-mode)

前者はモノレポ配下のパッケージを1つのバージョンで固定する方法。後者はパッケージ毎に任意のバージョンが指定可能。今回は、`Independent mode`を指定。

### ②について
[messageオプション](https://github.com/lerna/lerna/blob/main/commands/version/README.md#--message-msg)。`lerna version`をするときにlernaはコミットのバージョンをインクリメントして、CHANGELOG.mdを更新してコミットを作る。そのコミットに指定するメッセージを定義出来る。
Fixed/Locked modeであれば、メッセージに％sが含まれている場合は、「v」で始まる新しいグローバルバージョンのバージョン番号。％vが含まれている場合、先頭に「v」が付いていない新しいグローバルバージョンのバージョン番号に置き換えることが可能。
```json
{
  "command": {
    "version": {
      "message": "chore(release): publish %s"
    }
  }
}
```

後述のcommitlintを設定している場合、デフォルトだとルール違反と判定されて`lerna version`が失敗します。commitlintと併用する場合は設定が必要。

## commitlint
Conventional Commitsに従っているか人の目で確認するのは難しいため、commitlintを利用。
.commitlintrc.json
```json
{
  "extends": ["@commitlint/config-conventional"]
}
```
commitlintのルールは、[@commitlint/config-conventional](https://github.com/conventional-changelog/commitlint/tree/master/%40commitlint/config-conventional)の他にも[@commitlint/config-angular](https://github.com/conventional-changelog/commitlint/tree/master/%40commitlint/config-angular)がある。今のところ違いがよくわからないため、追加で調査したい。

## husky
コミットをする際に、commitlintとeslintが走るように設定する。
.huskyrc.json
```json
{
  "hooks": {
    "commit-msg": "commitlint -E HUSKY_GIT_PARAMS",
    "pre-commit": "yarn lint:fix"
  }
}
```

# コマンド関連
## パッケージの追加
共通利用するパッケージをインストールする。`W`オプションはYarn Workspaces内にパッケージの追加をすることを表しています。
```bash
yarn add -D -W @typescript-eslint/eslint-plugin
```

## ビルド
各パッケージで`yarn build`が実行されます。
```bash
lerna run build # yarn buildに設定するとよい
```

## バージョンアップ
`Independent mode`の場合、更新のあるパッケージのバージョンインクリメントをします。最初にバーションを対話式で選んだら、tag打ち、CHANGELOG.mdの更新をしコミットを打ってpushするとこまで自動で行います。

```bash
lerna version
```

## パッケージの公開
[publish](https://github.com/lerna/lerna/tree/main/commands/publish)コマンドを利用する
```bash
lerna publish              # 前回のリリース以降に変更されたパッケージを公開する
lerna publish from-git     # 現在のコミットでタグ付けされたパッケージを明示的に公開する
lerna publish from-package # 最新バージョンがレジストリ(npm)に存在しないパッケージを明示的に公開する
```
from-gitのユースケースは正直よくわからない。タグベースでリリースの公開制御するのはデメリットが大きいと考えている。例えば`leran version`は成功し、npmにログインしておらず、`leran publish`に失敗した場合にタグを削除しないとpublishすることが出来ないなど..。
from-packageを使えば漏れなくパッケージ公開可能なので、今回採用した。

## npm-scriptsにまとめる
npm-scriptsを利用し、新規リリースをする場合はビルドとバージョンアップとパッケージ公開は`yarn release`にまとめている。

```json
  "scripts": {
    "build": "lerna run build",
    "test": "lerna run build",
    "lerna:pub": "lerna publish from-package",
    "lerna:version": "lerna version",
    "release": "yarn build && yarn lerna:version && yarn lerna:pub",
    "lint:ts": "eslint --ext=.ts .",
    "lint:fix": "yarn lint:ts --fix"
  },
```

## CHANGELOGの抜き出し
特定のCHANGELOGを抜き出す
```bash
yarn monorepo-utils-collect-changelog --directory . "@sugoi-site/lib-a@0.4.0"
```

# おわりに
出来ていないこととして、GitHub ReleaseへCHANGELOGの書き出しがあり、ここは色々試してみる予定です。実際に個人ブログのCSSやマークダウンのパースライブラリを本記事の方法で管理します。その過程で使い辛い点は見直して、本記事に反映します!!

# 参考文献
[lernaでのmonorepoにおけるリリースフロー(Fixed/Independent)](https://efcl.info/2019/01/26/monorepo-release-flow/)
[fast-csv](https://github.com/C2FO/fast-csv)
