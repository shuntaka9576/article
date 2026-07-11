---
title: "Denoに入門する"
description: ""
publish: false
---

# はじめに
[denops.vim](https://github.com/vim-denops/denops.vim)が面白そうだったので、denoに入門することにした。

# 便利リンク

* [deno公式 | deno.land](https://deno.land/)

# インストール
公式参照。

# エディタ設定

denoのバージョンを確認
```bash
$ deno --version
deno 1.8.0 (release, x86_64-apple-darwin)
v8 9.0.257.3
typescript 4.2.2
```

[coc-deno](https://github.com/fannheyward/coc-deno)をインストール
```json: ~/.config/coc/extensions/package.json
{
  "dependencies": {
    "coc-css": ">=1.2.6",
    "coc-deno": ">=3.1.0", // ココ
    "coc-eslint": ">=1.4.5",
    "coc-flutter": ">=1.8.1",
    "coc-go": ">=0.13.3",
    "coc-html": ">=1.4.1",
    "coc-json": ">=1.3.4",
    "coc-python": ">=1.2.13",
    "coc-tsserver": ">=1.6.8",
    "coc-vimlsp": ">=0.11.1"
  }
}
```

# プロジェクト設定(coc.nvim)
tsserverがあると競合するのでfalseにする

```json: .vim/coc-settings.json
{
  "deno.enable": true,
  "deno.lint": false,
  "deno.unstable": true,
  "tsserver.enable": false
}
```

# 実行周り
## フォーマッターをかけつつ編集する

```ts
deno fmt --ext ts --unstable --watch
```

# 主要ライブライ

## log

```ts
$ export LOG_LEVEL=DEBUG

$ deno run -A ./src/logger/logger.ts
10 2021/03/14 00:42:28.022 Hello world
20 2021/03/14 00:42:28.024 123456
30 2021/03/14 00:42:28.024 true
```
