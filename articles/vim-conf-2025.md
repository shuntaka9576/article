---
title: "VimConf 2025 Smallに参加しました！"
description: "VimConf 2025 Small参加&簡易レポです"
publish: true
thumbnail: "https://res.cloudinary.com/dkerzyk09/image/upload/v1762071546/blog/vim-conf-2025/xsrcqgemtauiuor6hujm.png"
tags:
  - "tech/neovim"
  - "tech/イベント参加"
  - "tech/開発環境"
---

## はじめに

昨日(2025/11/02)[VimConf 2025](https://vimconf.org/2025/ja/)に参加しました！スケジュールは[こちら](https://vimconf.org/2025/ja/#schedule)

セッションは非常に充実した内容が多く、(Neo)Vimへのモチベーションが大きく上がる良い機会でした。

自分はプラグイン開発はしていなくて、Neovimユーザーという立場の小並感で申し訳ありませんが、各セッションの感想を書きます！

:::message warning
個人の解釈を含みます。誤っている場合、以下までご連絡ください 🙇

https://github.com/shuntaka-dev/article/issues
:::

## セッションの感想

### nvim-cmp retrospective: Exploring Completion and Facing FOSS Challenges

nvim-cmpは自分もよく使っていたので、とても気になっていた話。

LSPは基本的な仕様のみを定めており、細かいUXの部分は各エディタの実装次第という部分が多いんだなと感じた。またユーザーに提供したいUXを実現するために、LSPから補完プラグイン、Vimとの間で猛烈な腕力で調整を重ねている実態が垣間見えた。LSPが提供しないpreview textをヒューリスティックな実装で実現するなど。補完候補の選択時挿入など、特別意識していなかったが、なくなると違和感を感じる部分で大事だなぁと思った。リファレンス実装としてVSCodeの実装を確認するところも参考になりました。

後半では、9,000+ starsという予想外の人気により、OSSメンテナとしての葛藤が語られていた。安定的な運用を目指すために貢献者を増やしたくても、信用できる貢献者を探すのが大変。サプライチェーン攻撃などリスクなどもある。(SpaceVimの作者の方は活動を知っていることもあり、コラボレータとして手伝って貰っているという話がセッション後の質疑の回答であった。いい話。)。質疑であった補完エンジンの歴史、現在はblink.cmpはRust実装が一部あってFFIで高速化している...という話も気になった。

技術的な深さだけでなく、FOSSの責任の重さについても考えさせられるセッションでした！

### スポンサーLT (エンジニアの楽園 vim-jpラジオ)

vim-jpラジオ聞いてます！いつもありがとうございます！！！

### Introducing the TabPanel Feature: Motivation and Implementation

TabPanelの機能紹介と、Vim本体への取りこみ。この機能を入れたforkを5年メンテしていたというのが驚きだった。質疑で、Bram時代にVimのソースコードリファクタリングをしていたときの追従(コンフリクト解消)が大変だったというお話で、本体に取り込まれて良かったと思った。

あっさりマージされたと語られていたが、5年ドッグフーディングしていたという見方も出来るような気がしていて、筋の良さがあったのかなと感じた。thincaさんの[ポスト](https://x.com/thinca/status/1984812986931036524)からもスムーズだった理由がポストされていました！


### Lunch Break

ランチは今年はなかったので、近くの吉野家で食べました 😋


### Lowering the Vim Barrier: Building nvim Environment with AI Assistance

生成AIでVimの障壁を超える話。

知らないプラグインも多くて、3ヶ月のインプットとしてはすごいなと感じた。

セッションとは関連性が薄いですが、昨日Omarchyを入れて[lazy.nvim](https://github.com/folke/lazy.nvim)がプリセットで入っていて、補完のカスタマイズが良くて自分でメンテが追いつかないならこういうのを頼ってもいいなと思っている今日この頃でした。

### Getting Started with *your own* Neovim feat. mini.nvim

mini.nvim 40以上のモジュールがプリセットされているプラグイン。プラグインがある世界でのNeovimの機能概念が分かる良いプラグインだと感じた。Vimはまず始める段階から、プラグインをたくさん使ってIDEみたいにするまでに挫折ポイントが多いと感じていたので、mini.nvimレイヤーはとても良いと思った。。自分の設定はにわか知識で継ぎ足しで文字通り秘伝のタレになっているので、プリセットの方が自作の設定より、効率的で概念を理解しやすいので、自分の設定をまっさらにしてmini.nvimからやり直したいと思った。

### And Yet, Vim Survived: Thinking and Seeing in the Age of Code You Don't Write

Vimをコードを読むためのツールとして使う際のTipsのお話。

コンピューターがコードを書いて、人間がチェックする時代。今は沢山早く書くより、より深く理解することが大事。コードの追いかけ方としてWhere(流れ, Flow) What(構造, Structure) Why(理由, Reasoning)が話していた。

Where(Flow)では以下はターミナルでエラー発生した場合、その場所にジャンプ、戻る方法で`lambdalisue/vim-file-protocol` `lambdalisue/vim-gf-improved` `Backudankun/BackAndForward.nvim`を使う。fzf(`fall.vim`)を使って、ピンポイントでジャンプして、戻る方法など。

What(Structure)ではウィンドウ分割の話、`junegunn/goyo.vim` も使っているとのこと。分割にもキーマッピングを設定している感じ(？) `:sp`と:`vs`しか使ってなかったので参考になった。フォーカスの当て方も便利そうでした。プロジェクトレベルではvim-fernとfall.vimの紹介。

Whyでは、Ginの紹介。`:GinStatus` `:GinLog` `:GinBlame` `:GinDiff` `:GinLog`の紹介。lazygitで甘えているのでGin使いこなしたい。

キーバインディングの設定も乗っていて、後から見直して参考にしたいと感じました！

### スポンサーLT (株式会社テックリード)

ゴリラさん毎年スポンサー感謝！

### Beyond Syntax Highlighting: Unlocking the Power of Tree-sitter in Neovim

Tree-sitterの話。

urlを認識してくれたり、マークダウンのように他の言語が混ざっていても再帰的にパースできる。`query` `parser` `tree-traversal` `callback`の機能に関する話、プラグインとして`vim-matchup` `nvim-treesitter-context` `nvim_context_vt` の紹介があった。

スピーカーの方が実装した[treemonkey.nvim](https://blog.atusy.net/2023/12/27/treemonkey-nvim/)はTree-sitterでパースしてセマンティクスにあった選択ができるようになるみたい👀 すごい。[treesitter-ls](https://github.com/atusy/treesitter-ls)は複数言語にわたる正確な解析ができるLSPの紹介もありました！構文情報が取れるので、色んな活用ができそう？ASTとは違うレベルなのかな。とりあえずここら辺必要になってみたら試していきたいなと思いました！

### Assists text writing anywhere

nvim-aiboやeditpromptを使ってコーディングエージェントへの入力するデモが便利そうでした！Vim外でもテキストエリアでVimできるソリューションありがたいですね...。alisueさんのセッションでClaude CodeでSKKしていたのが気になっていたので、nvim-aiboで出来そうということが分かって良かった。

### Designing Repeatable Edits: The Architecture of . in Vim

dot repeatの話。

Vimのプリミティブな機能の話。dot repeatは最後のundoブロックを再実行する機能。繰り返し可能な作業に落とし込むのが大事そう。 dot repeatはglobalに適用することも可能(`g/{pattern}/normal .`)。自分の作業をdotに最適化したい...

### You Know They're Not Just Clipboards — Now Learn What They Really Are

レジスタまわりや<C-r>の使い方など知らないことだらけでした。。。

基礎的なVimの話、私は知識がなく、学び直したい...表がとてもみやすく、改めて手を動かしながら読み返したいありがたい2つのお話でした！


### LTs

* LT1
  * pytorchの前身がlua実装？torchをNeovimから呼び出せるっぽい。知見💡
  * notomo/gesture.nvim
* LT2
  * vim9Scriptの話
  * 最新のVimが必要
  * 高速だけどLSPがないので辛い部分もあるとか
  * 開発中の機能もある
* LT3
  * unit test
    * plenary.test_harness
      * みんな入っている
      * neotest
      * zennに記事がある
    * vusted mini.test
      * lua-covと連携できる
      * カバレッジが測れる
      * Speakerは両方(plenary)使っているみたい
        * plenaryと貴方は同じなので、plenaryで書いて移行も可能
    * octcov
  * mini.test
    * E2E
      * 非同期のテストが難しいので、こちらがおすすめ
      * neovimを子プロセスとして動作させる
      * スクリーンショットもできる
      * カバレッジは低いのではなく伸び代がある
      * octocovが便利
* LT4
  * Vimの中で休憩する話
    * ボードゲームをする
    * Vimの中でリバーシする
    * ゼロ依存で実装するために、アルゴリズムに凝っていて面白い...
* LT5
  * Fluent finderの話
  * [flf-vim](https://github.com/sirasagi62/flf-vim)
  * ソースに対して意味的類似性検索ができるみたい！🤩🤩🤩
  * インデックス化が1時間かかるみたい？
  * 埋め込みモデルの話もあった
    * SLM ruri-v3-70-code-v0.1
    * 多分これかなぁ
      * https://huggingface.co/cl-nagoya/ruri-v3-70m
    * plamoと比較してどうなのかな、外部依存とかどうなのかな。気になる👀
    * おもしろい取り組み⭐️
* LT6
  * Keymapping
  * nvim-in-the-loop
  * 自分のVimの操作のログをとって、AIに食わせて癖を見つける。AIに良いマッピングを提案してもらう🤩

## さいごに

補完プラグイン開発、本体開発の話から始まり、ソースコードの読み方、CLIのコーディングエージェントとの連携、Vimを取り巻くエコシステムや、Vimの基礎的な話など、今年も変わらない基礎から最近の(Neo)Vimの知識まで濃厚に注入できました！

スポンサー、運営の皆様ありがとうございました！✨
