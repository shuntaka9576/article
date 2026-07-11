---
title: "Rustのコンパイルエラーまとめ"
description: "Rust学習中で出会ったコンパイルエラーをまとめます"
publish: true
tags:
  - "tech/rust"
  - "tech/開発環境"
---

# はじめに

事象には、以下の記載する

* タイトルは主要なエラーメッセージ内容
* エラーが発生する最小限コード
* コンパイル結果

自分はRust初心者で、随時更新していきます。かなり簡単な内容も書いていきます。

# エラー例

## cannot use the `?` operator in a function that returns `()`

### 事象

```rust
use rustyline::Editor;

fn main() {
    let mut rl = Editor::<()>::new()?;
}
```

```bash
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:4:37
  |
3 | fn main() {
  | --------- this function should return `Result` or `Option` to accept `?`
4 |     let mut rl = Editor::<()>::new()?;
  |                                     ^ cannot use the `?` operator in a function that returns `()`
  |
  = help: the trait `FromResidual<Result<Infallible, ReadlineError>>` is not implemented for `()`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `s007-my` (bin "s007-my") due to previous error
[Finished running. Exit status: 101]
```

### 調査

* `?`演算子は、`Result`または`Option`を返す関数内でしか利用できない
* `?` 演算子は、「エラーがあったらreturnしてそうでないなら中の値が欲しい」ケースで便利
* `?`演算子はエラーを伝播する。今回は`new()?`でエラーが発生した場合、それを伝播するため、main関数の`()`(=タプル型)は一致しないことになる。



### 解決策

```rust
use rustyline::Editor;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let mut rl = Editor::<()>::new()?;

    Ok(())
}
```

### 余談

#### Try Trait v2

* [【Rust RFC 3058: Try Trait v2】?演算子（Question Operator）の新しい挙動についてのまとめ](https://xbgneb0083.hatenablog.com/entry/2022_7_22_try_trait_v2)が参考になる。

以下のコードで`ret_ok`が`print_if_ok`で、`?`演算子をつけて呼び出されていることから、Result型を返却する作りになっている。
```rust
fn ret_ok() -> Result<isize, ()> {
    Ok(10)
}

fn print_if_ok() -> Result<(), ()> {
    println!("{}", ret_ok()?);

    println!("print succeed");
    Ok(())
}
```

`Try Trait v2`はこれを明確な型を返却するように定義することが可能。


### 参考
* Bing Chat
* [Stabilize main with non-() return types](https://github.com/rust-lang/rust/issues/48453)
