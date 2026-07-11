---
title: "新しいJavaScriptランタイムBunとその開発言語Zigの開発環境を作る"
description: "Neovim coc.nvimでLSP設定までわりとサクッとできました"
thumbnail: ""
publish: true
tags:
  - "tech/neovim"
  - "tech/zig"
  - "tech/typescript"
  - "tech/開発環境"
---

# はじめに

最近新しいJavaScriptランタイム[Bun](https://bun.sh/)がが盛り上がっているみたいなので、Bunとその開発言語であるZigの開発環境構築をしてみました。
公式以上の内容はないので、Macで概要だけ掴みたい人もしくは、Neovim/coc.nvim構成でzigの開発環境を構築したい人向けです。

# そもそも

JavaScriptランタイムでは、denoが最近では有名であり、RustでV8 bindingで実装されています。一方Bunは、JavaScriptCoreをZigという言語で拡張しているみたいです。

# Bun

## インストール

```bash
$ curl -fsSL https://bun.sh/install | bash

######################################################################## 100.0%
Bun was installed successfully to ~/.bun/bin/bun

 Added "~/.bun/bin" to $PATH in "~/.zshrc"

To get started, run

   exec /bin/zsh
   bun --help
```

インストールすると、`~/.zshrc`に自動で追記されます。

```bash:~/.zshrc
[ -s "$HOME/.bun/_bun" ] && source "$HOME/.bun/_bun"
```

パスを通す
```bash:~/.zshenv
export PATH="$HOME/.bun/bin:$PATH"
```

### 実行

内容的にasyncする意味はないですが、、トップレベルawaitできる確認する意味で、、
```ts:main.ts
const main = async () => {
  console.log("hello world!");
};

await main();
```

```bash:実行
$ bun main.ts
hello world!
```


# Zig 

## インストール

[ziglang.org](https://ziglang.org/learn/getting-started/) 参照

```bash
brew install zig

# latest
brew install zig --HEAD
```

## 実行

```bash
$ zig --help
Usage: zig [command] [options]

Commands:

  build            Build project from build.zig
  init-exe         Initialize a `zig build` application in the cwd
  init-lib         Initialize a `zig build` library in the cwd

  ast-check        Look for simple compile errors in any set of files
  build-exe        Create executable from source or object files
  build-lib        Create library from source or object files
  build-obj        Create object from source or object files
  fmt              Reformat Zig source into canonical form
  run              Create executable and run immediately
  test             Create and run a test build
  translate-c      Convert C code to Zig code

  ar               Use Zig as a drop-in archiver
  cc               Use Zig as a drop-in C compiler
  c++              Use Zig as a drop-in C++ compiler
  dlltool          Use Zig as a drop-in dlltool.exe
  lib              Use Zig as a drop-in lib.exe
  ranlib           Use Zig as a drop-in ranlib

  env              Print lib path, std path, cache directory, and version
  help             Print this help and exit
  libc             Display native libc paths file or validate one
  targets          List available compilation targets
  version          Print version number and exit
  zen              Print Zen of Zig and exit

General Options:

  -h, --help       Print command-specific usage
```

```bash
$ mkdir hello-world
$ cd hello-world
$ zig init-exe
info: Created build.zig
info: Created src/main.zig
info: Next, try `zig build --help` or `zig build run`
$ tree
.
├── build.zig
└── src
    └── main.zig
$ zig build run
info: All your codebase are belong to us.
```

# Neovim

## LSP設定(coc.nvim)

cocプラグインを導入
```vim
:CocInstall coc-zig
```

LSPの[zls](https://github.com/zigtools/zls)のREADMEをみつつ、導入

```bash
brew install xz
mkdir $HOME/zls && cd $HOME/zls && curl -L https://github.com/zigtools/zls/releases/download/0.9.0/x86_64-macos.tar.xz | tar -xJ --strip-components=1 -C .
```

実行権限を付与
```bash
$ chomd +x ~/zls/zls
```

パスを通す
```bash:~/.zshenv
export PATH="$HOME/zls:$PATH" # LSP
```

`coc-settings.json`にzlsのパスを追記

```json:~/.config/nvim/coc-settings.json
  "zig.path": "~/zls/zls"
```

補完がバシバシ効きますし、コードジャンプも快適です。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1657441768/blog/01g7kk2jy54b9d7ct876mexwkm/lp9pgnnhufemxkfozpcn.png)

## その他(tabやオートフォーマット設定)

公式が出しているVimプラグイン[ziglang/zig.vim](https://github.com/ziglang/zig.vim)を利用。init.luaの詳細は、[shuntaka9576/dotfiles](https://github.com/shuntaka9576/dotfiles/blob/master/lua/init.lua)参照。

```lua:init.lua設定
vim.api.nvim_command("autocmd BufNewFile,BufRead *.zig setlocal tabstop=4 shiftwidth=4 expandtab")
...
use({
  "ziglang/zig.vim",
  function()
    vim.cmd [[
      let g:zig_fmt_autosave = 1 -- オートフォーマット設定を有効化(便利なので特別な理由がない限り有効化推奨)
    ]]
  end,
})
```




# さいごに

あまり環境構築周りでハマることはなかったです、LSPやオートフォーマッターも揃っていていい感じです。WASM周りが気になるので、別記事で試していこうと思います。


