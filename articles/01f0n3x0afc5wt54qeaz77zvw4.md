---
title: "Vim/Neovimプラグイン開発サイクルをまとめてみる"
type: "tech"
description: "denops.vimでプラグイン開発するにあたり、初心者なりに開発サイクルを考えてみました。"
publish: true
tags:
  - "tech/neovim"
  - "tech/typescript"
  - "tech/開発環境"
---

# はじめに
[denops.vim](https://zenn.dev/lambdalisue/articles/b4a31fba0b1ce95104c9)が面白そうで触り始めた。その過程で私自身プラグインを作ったことはあるものの、改めてプラグイン開発者はどんなサイクルで動作確認しているのかと疑問に感じた。なので自分で開発するならこうするしかないなーと思うサイクルを書くことにした。正直あまり変わったことはしていない。。


# 前提
以下を前提とした開発サイクルとする

* Vim/Neovimを利用
* denops.vimを利用したプラグイン開発
* vimscriptを利用したプラグイン開発

# 実現したいこと
* 開発で利用するVim/Neovimの設定に影響を与えないこと

# Vim/Neovimの環境設定

|Vim/Neovim|version|
|---|---|
|Vim|VIM - Vi IMproved 8.2 |
|Neovim|NVIM v0.5.0-828-g0a95549d6|


|設定|開発するVim/Neovim|プラグインの動確をするVim/Neovim|
|---|---|---|
|init.vim|~/.config/nvim/init.vim|`vim -S` `nvim -u`で任意指定|


# 最低限のinit.vim
動作検証のため、副作用のないinit.vimを作ります。私はリポジトリを作って管理している。

下記のような仮定に基づいた`init.vim`を示す
* vimscriptで開発中のプラグイン名を`practice.vim`とする
* denops.vimエンジンを使った開発中のプラグイン名を`dps-helloworld`とする

```vim:~/repos/github.com/shuntaka9576/init.vim/init.vim
" --- denopsエンジンの読み込み ---
set runtimepath+=~/repos/github.com/vim-denops/denops.vim

" --- 開発するプラグインの読み込み ---
" Vimの場合,autoloadの遅延ロードのみ
"   $runtimepath/plugin/**/*.vimを読み込むため
" Neovimの場合,autoloadの遅延ロード,pluginのロード
set runtimepath+=~/repos/github.com/shuntaka9576/practice.vim
set runtimepath+=~/repos/github.com/shuntaka9576/dps-helloworld

" --- vimのみ ---
if !has('nvim')
  " -uコマンドを実行するとcompatibleオプションがONになる
  " vi互換とか持たせるためのオプション(?)
  set nocompatible
  " Vimのpluginのロード
  source ~/repos/github.com/vim-denops/denops.vim/plugin/denops.vim
  source ~/repos/github.com/shuntaka9576/dps-helloworld/plugin/deps-helloworld.vim
  source ~/repos/github.com/shuntaka9576/practice.vim/plugin/practice.vim
endif

" --- 追加設定 ---
" Vimのファイルタイプ検出,ファイルタイププラグインとインデントプラグインをオンにする設定
filetype plugin indent on

" --- denopsの設定 ---
" Denoの起動時型チェックを有効化(開発が安定したら明示指定を削除する
let g:denops#server#service#deno_args = [
      \ '-q',
      \ '--unstable',
      \ '-A',
      \ ]
```

`Vim` `Neovim`に前述のinit.vimを指定して、実行します

```bash:Vimの場合
vim -u ~/repos/github.com/shuntaka9576/init.vim/init.vim
```

```bash:Neovimの場合
nvim -u ~/repos/github.com/shuntaka9576/init.vim/init.vim
```



# 開発の流れ

## 最初にやること
* [#最低限のinit.vim](#最低限のinit.vim)に開発するプラグインのパスを追記
* coc.nvimを利用している場合は、denoの設定ファイルを作成

```json:.vim/coc-settings.json
{
  "deno.enable": true,
  "deno.lint": false,
  "deno.unstable": true,
  "tsserver.enable": false
}
```

## 開発中
1. プラグインを改修
2. `vim -u` `nvim -u` を利用して動作確認
3. プラグインを改修
4. 2で起動したVim/Neovimを終了し、再起動して動作確認

# 最後に

本記事に関して誤っている点やもっと良い方法があれば[issue](https://github.com/hozi-dev/article/issues)で受け付けています！以上。
