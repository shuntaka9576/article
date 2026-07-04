---
title: "iamcco/markdown-preview.nvimのソースリーディング"
type: "tech"
description: ""
publish: false
---

# markdown-preview.nvimとは
マークダウンをリアルタイムでプレビュー可能なプラグイン。Neovimのリモートプラグインを仕組みを使っています。リモートプラグインを開発する上で必要なVim Scriptのロジックを知る目的があります。

# ソースコード概要
```bash
$ git rev-parse --short HEAD
e5bfe9b

$ tokei
-------------------------------------------------------------------------------
 Language            Files        Lines         Code     Comments       Blanks
-------------------------------------------------------------------------------
 Batch                   1           29           19            0           10
 CSS                     5          497          454           12           31
 HTML                    2            2            2            0            0
 JavaScript             55         2982         2400          339          243
 JSON                    8          949          949            0            0
 JSX                     2          334          286           21           27
 Markdown                3          696          696            0            0
 Shell                   2          136          105            9           22
 TypeScript             24          462          389           40           33
 Vim Script              8          994          864           21          109
-------------------------------------------------------------------------------
 Total                 110         7081         6164          442          475
-------------------------------------------------------------------------------
```
結果をみると、言語的に支配的なのJavaScriptであることが分かります。Vim Scriptは994行程となっています。


# Vim Script

```bash
├── autoload
│   ├── health
│   │   └── mkdp.vim
│   └── mkdp
│       ├── autocmd.vim
│       ├── nvim
│       │   ├── api.vim
│       │   └── rpc.vim
│       ├── rpc.vim
│       └── util.vim
├── plugin
    └── mkdp.vim
```

ディレクトリ構成と、各ファイルの行数は下記の通り。
|ディレクトリ|フィル名|行数|
|---|---|---|
|/autoload/health|mkdp.vim|19|
|/autoload/mkdp|autocmd.vim|22|
|"|util.vim|185|
|"|rpc.vim|167|
|/autoload/mkdp/nvim|api.vim|327|
|"|rpc.vim|133|
|/plugin|mkdp.vim|129|
|合計||982(※)|

※ テストのinit.vimを除いたため、-12した982行となる
