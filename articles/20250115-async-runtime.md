---
title: "言語ランタイムと並行実行/並列実行"
type: "note"
description: ""
thumbnail: ""
publish: true
tags:
  - "tech/rust"
  - "tech/go"
  - "tech/node.js"
  - "tech/技術メモ"
---


## Rust

Rustは標準で非同期ランタイムが組み込まれていない。代わりに、非同期プログラミングのための基盤となる以下のような機能を提供してる。

* Future trait - 非同期処理の抽象化
* async/await 構文 - 非同期コードを書くための構文糖。非同期のコンテキストに入るためにsyntaxとして提供されている。
* Pin 型 - 自己参照構造体のための型

非同期関数は必ずランタイムによって実行されないければならない。

ランタイムを構成する要素は以下の通り

* 非同期タスクのスケジューラ
* イベント駆動のI/O
* タイマー


### Tokio

[RustのTokioで非同期とグリーンスレッドを理解する](https://zenn.dev/tfutada/articles/5e87d6e7131e8e)より

> 乱暴な言い方をすれば、Node.jsとGolangを合体させたのがtokioです。
> Node.jsとはC10Kのような膨大なリクエストをシングルスレッド&イベントループ&非同期I/Oで効率良く処理するランタイムです。実際、Denoにはtokioが使用されています。tokioでもasync/awaitを使用して非同期プログラミングをします。
> Pythonでもaysncioのような非同期プログラミングが導入されています。Kotlinにもコルーチンというグリーンスレッドとasync/awaitを使用した非同期プログラミングがあります。


このNode.jsとGoを合体させたという表現が言い得て妙です。tokioは非同期IOとグリーンスレッドの合わせ技というところでしょうか。

グリーンスレッドを使う場合は、こんかイメージでしょうか。(ここら辺は理解の深掘りが必要)
* I/Oの非同期処理自体は Node.js と似た方式
* タスクの実行管理は グリーンスレッド方式

```rust
// グリーンスレッドとして実行
tokio::spawn(async {
    // 非同期タスク
});
```

* Tokio ランタイム上でタスクを spawn するとき、その型は 'static の必要がある。spawn されるタスクは、タスクの外で所有されているデータへの参照を含んではいけない。
* Send境界。spawn されるタスクは 必ず Send トレイトを実装している必要がある。

これは所有権の仕組み上そうだよねって感じ。

CPUバウンド処理を行う場合、`spawn_blocking`を使ってTokio経由でOSスレッドを使う必要がある。

```rust
// spawn_blockingでOSスレッドプールで実行
tokio::task::spawn_blocking(|| {
    // CPU集中型の処理
    heavy_computation();
});
```


## Go

Go自体にgoroutineというグリーンスレッド実装が組み込まれている。あるゴルーチン（G）がI/Oの完了を待っているとき、そのゴルーチンを一時的に停止させ、代わりに別の実行待ちのゴルーチンを動かすことができる仕組みが備わっています。これにより、CPUの処理能力を効率的に使うことができます。

[プリエンプション](https://zenn.dev/hsaki/books/golang-concurrency/viewer/gointernal#%E5%AE%9F%E8%A1%8C%E3%82%B4%E3%83%BC%E3%83%AB%E3%83%BC%E3%83%81%E3%83%B3%E3%81%AE%E3%83%97%E3%83%AA%E3%82%A8%E3%83%B3%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3)より

非同期IO


## Node.js

Node.jsはシングルスレッドのイベントループ。libuvが利用されている。

## Deno

Tokioを利用。


## Elixir/Erlang

最重要コンセプトは並行指向プログラミングらしい。Goと似ていて、グリーンスレッドとプリエンプティブなスケジュールが可能。

## 用語

### プリエンプション
実行中のタスクを一時的に中断して、他のタスクを実行させること


### 後で調べる

* [チャイルドスティーリング・継続スティーリングについて](https://zenn.dev/nasa/articles/compare_rust_go_concurrency#%E3%82%BF%E3%82%B9%E3%82%AF%E3%82%B9%E3%82%B1%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AA%E3%83%B3%E3%82%B0%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)
