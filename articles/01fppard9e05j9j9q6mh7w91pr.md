---
title: "wasmライブラリをlernaでnpmに公開する構成"
description: "wasmパッケージをnpmに公開する際に嵌ったことを書きます"
publish: true
tags:
  - "tech/rust"
  - "tech/wasm"
  - "tech/npm"
  - "tech/モノレポ"
---

# はじめに
簡単な数値計算ライブラリはRustで書いて、wasmをフロント側で使うという利用例がこれから増えてくると思っています。(要出典)
wasm-packでビルドした資材をnpmに置いておけば、フロント側はnpmなりyarnなりで落として使えて便利だなと思いlernaと合わせて試してみました。

::: message warning
どちらかというとnpm publish周りの挙動に関する記事で、Rustとかwasmあまり関係ないです。紛らわしいタイトルで申し訳ありません。
:::

# ディレクトリ構成

ディレクトリ構成は、モノレポ構成をとります。今回は`@shuntaka/poc`という名前でnpmパッケージをあげることを仮定します。packages配下の階層にパッケージが複数並ぶイメージです。

```bash
├── README.md
├── lerna.json
├── package.json
├── packages/poc
│   ├── CHANGELOG.md
│   ├── Cargo.toml
│   ├── LICENSE_APACHE
│   ├── LICENSE_MIT
│   ├── README.md
│   ├── package.json
│   ├── src
│   │   ├── lib.rs
│   │   └── utils.rs
│   └── tests
│       └── web.rs
└── yarn.lock
```


# wasmテンプレートの作成

```bash
mkdir packages
cd packages
```

wasmテンプレートをジェネレートします
```bash
cargo generate --git https://github.com/rustwasm/wasm-pack-template
# 対話型でプロジェクト名の入力を求められる -> pocと入力
```

packages.jsonファイルを追加します
```json:packages/poc/packages.json
{
  "name": "@shuntaka/poc",
  "version": "0.0.1",
  "description": "poc lib",
  "prepublishOnly": "yarn build && cp -r ./pkg/* . && rm -rf ./pkg",
  "publishConfig": {
    "access": "public"
  },
  "files": [
    "poc*"
  ],
  "scripts": {
    "build": "wasm-pack build",
    "prepublishOnly": "yarn build && cp ./pkg/poc* .",
    "postpublish": "git clean -fd"
  },
  "main": "poc.js",
  "types": "poc.d.ts"
}
```
本設定ファイルのポイントについて解説します

## パッケージ名のエイリアス
`name`フィールドは、npm installする際に指定するパッケージ名となります。`@shuntaka`のようなエイリアスを設定する場合は、事前にnpmのWebコンソールでOrganizationsを設定する必要があります。

## npmパッケージを作る上で注意点
`pkg`ディレクトリは、`wasm-pack`のビルド資材が格納されるディレクトリです。`package.json`のfilesオプションで、pkgディレクトリを指定することが可能ですが、その場合npmパッケージ側に`pkg`というディレクトリ階層が含まれてしまいます。
```bash
@shuntaka/poc/pkg/[ビルド資材]
```

今回は`@shuntaka/poc/[ビルド資材]`という構成にしたいため、`prepublishOnly`でビルド資材をpocプロジェクトルートにcpしています。その後、gitのコマンドでuntrackなファイルを削除しています。
npmへpublishされるファイルが何かイメージがつきにくい場合は、`npm pack --dry-run`を使うとよいです。(prepublishOnlyは実行されないので手動実行が必要です。)

pkgディレクトリで、`npm publish`をする方法もありますが、今回lernaと合わせて使う要件があったため見送りました。

本方法は、[stack overflow | How to npm publish specific folder but as package root](https://stackoverflow.com/questions/38935176/how-to-npm-publish-specific-folder-but-as-package-root)を参考にしました。


# モノレポ設定
lernaの設定に関しては、割愛します。特別なことはしていません。

```json:lerna.json
{
  "npmClient": "yarn",
  "useWorkspaces": true,
  "packages": [
    "packages/*"
  ],
  "command": {
    "version": {
      "message": "chore(release): publish",
      "conventionalCommits": true
    }
  },
  "version": "independent"
}
```

```json:packages.json
{
  "name": "root",
  "scripts": {
    "lerna:pub": "lerna publish from-package",
    "lerna:version": "lerna version",
    "release": "yarn lerna:version && yarn lerna:pub"
  },
  "private": true,
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "nohoist": []
  },
  "devDependencies": {
    "lerna": "4.0.0"
  }
}
```


# リリースする

## ソースを修正してコミット
alertの内容を修正します。コミットメッセージは、Conventional Commitsでバージョンが変わるprefixを利用してください。Conventional Commitsについては、[npmモジュールをモノレポで管理し、煩雑なリリース業務を効率化する](https://shuntaka.dev/shuntaka/articles/01evbw029qzxavp20erstgvm5r#conventional-commits)で詳しく解説しております。参考まで。

```rust:lib.rs
mod utils;

use wasm_bindgen::prelude::*;

// When the `wee_alloc` feature is enabled, use `wee_alloc` as the global
// allocator.
#[cfg(feature = "wee_alloc")]
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet() {
    alert("Hello, poc! 12/12 12:08"); // <-- 修正
}
```


## リリース
`yarn release`でreleaseできます

:::details ログ(※ 本ログは記事向けに実際の出力をそれっぽく改竄しています)

```bash
$ yarn release
yarn run v1.22.15
$ yarn lerna:version && yarn lerna:pub
$ lerna version
lerna notice cli v4.0.0
lerna info versioning independent
lerna info Looking for changed packages since @shuntaka/poc@0.0.15
lerna info getChangelogConfig Successfully resolved preset "conventional-changelog-angular"

Changes:
 - @shuntaka/poc: 0.0.1 => 0.0.2

? Are you sure you want to create these versions? Yes
lerna info execute Skipping releases
lerna info git Pushing tags...
lerna success version finished
$ lerna publish from-package
lerna notice cli v4.0.0
lerna info versioning independent
lerna WARN Yarn's registry proxy is broken, replacing with public npm registry
lerna WARN If you don't have an npm token, you should exit and run `npm login`

Found 1 package to publish:
 - @shuntaka/poc: 0.0.1 => 0.0.2

? Are you sure you want to publish these packages? Yes
lerna info publish Publishing packages to npm...
(中略)

$ wasm-pack build
[INFO]: 🎯  Checking for the Wasm target...
[INFO]: 🌀  Compiling to Wasm...
   Compiling poc v0.1.0 (/Users/takahashi.shunichi/repos/github.com/shuntaka/gssim-packages/packages/poc)
warning: function is never used: `set_panic_hook`
 --> src/utils.rs:1:8
  |
1 | pub fn set_panic_hook() {
  |        ^^^^^^^^^^^^^^
  |
  = note: `#[warn(dead_code)]` on by default

warning: `poc` (lib) generated 1 warning
    Finished release [optimized] target(s) in 0.85s
[INFO]: ⬇️  Installing wasm-bindgen...
[INFO]: Optimizing wasm binaries with `wasm-opt`...
[INFO]: Optional fields missing from Cargo.toml: 'description', 'repository', and 'license'. These are not necessary, but recommended
[INFO]: ✨   Done in 1.26s
(中略)

> git clean -fd

Removing poc.d.ts
Removing poc.js
Removing poc_bg.js
Removing poc_bg.wasm
Removing poc_bg.wasm.d.ts
lerna success published @shuntaka/poc 0.0.16
lerna notice
lerna notice 📦  @shuntaka/poc@0.0.16
lerna notice === Tarball Contents ===
lerna notice 818B poc_bg.js
lerna notice 67B  poc.js
lerna notice 482B package.json
lerna notice 47B  README.md
lerna notice 114B poc_bg.wasm.d.ts
lerna notice 80B  poc.d.ts
lerna notice 279B poc_bg.wasm
lerna notice === Tarball Details ===
lerna notice name:          @shuntaka/poc
lerna notice version:       0.0.16
lerna notice filename:      shuntaka-poc-0.0.16.tgz
lerna notice package size:  1.2 kB
lerna notice unpacked size: 1.9 kB
lerna notice shasum:        bbdc51558495e9aedfbe203bffc22fcaee314af3
lerna notice integrity:     sha512-YkL7TXgQ9jXBh[...]J2/BuVo9xbT2g==
lerna notice total files:   7
lerna notice
Successfully published:
 - @shuntaka/poc@0.0.16
lerna success published 1 package
✨  Done in 14.53s.
```
:::


# フロント側で利用する

テンプレートを作成します
```bash
npm init wasm-app www
cd www
npm install @shuntaka/poc
```

`index.js` 書き換えます
```js:index.json
-import * as wasm from "hello-wasm-pack";
+import * as wasm from "@shuntaka/poc";
```

```bash
yarn start
```

修正されたwasmのnpmパッケージが動作していることを確認できました。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1639282972/blog/01fppard9e05j9j9q6mh7w91pr/hsvrq0tyu9zjwxzohqme.png)

# 最後に
本方法を用いれば、wasmライブラリ側とフロント側を疎結合なリポジトリ構成で構成できるのではないでしょうか。Rustで書いてフロントで使うのはなかなか不思議な感覚で、楽しかったです。

これから改善していく過程で発見がありましたら、また記事にしようと思います。

# 参考資料
[Rust and WebAssembly](https://rustwasm.github.io/book/introduction.html#rust--and-webassembly-)

