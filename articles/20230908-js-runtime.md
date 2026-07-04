---
title: "JavaScriptランタイムの違いをまとめる"
type: "note"
description: ""
publish: true
tags:
  - "tech/javascript"
  - "tech/node.js"
  - "tech/deno"
---


## はじめに

昨今JavaScriptランタイムがいくつか出てきており、それぞれ違いがあるので、まとめてみることにする


:::message
この手の解説は、図にするがベストなので余裕があるときにやります、、
古川さんの資料が本当に分かりやすく。その引用ばかりになります。。ご了承ください。
:::


## 前提

### JavaScriptエンジン

* V8
* JavaScriptCore(Nitro)


### JITコンパイラ

JITコンパイルは、プログラム実行時にソースコードや中間コードをbytecodeへ変換します。1行1行実行するインタプリタと比較して、コンパイル時のオーバーヘッドがあるが、実行時の速度が速い。

V8はインタプリタを使わず、JITコンパイラでJavaScriptをより効率が良い機械語に変換します。




### 言語処理方法

1. Tokenize(ソースコードから、Token)
2. Parse(Tokenから、AST)
3. Compile(ASTから、Byte Code)
4. Byte Codeから機械語

|英語|日本語|
|---|---|
|Tokenize|字句解析|
|Parse|構文解析|
|Token|トークン列|
|AST|抽象構文木|

@[sd](2ac6cd2798264833af10d9b36c36a79c,35,560/420,1.3)


## Node.js

### 構成

@[sd](2ac6cd2798264833af10d9b36c36a79c,21,560/420,1.3)


## Bun

### 構成

@[sd](2ac6cd2798264833af10d9b36c36a79c,22,560/420,1.3)



## 参考

[JavaScriptエンジンでの処理を理解する](https://zenn.dev/oreo2990/articles/5b56230ba2d7a1)
[Bunファーストインプレッション - JavaScriptランタイム界に”赤壁の戦い”を!](https://gihyo.jp/article/2023/01/tfen005-bun)
[Bun first impressions](https://speakerdeck.com/yosuke_furukawa/bun-first-impressions-techfeed)
