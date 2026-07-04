---
title: "VimConf 2024に参加しました！"
type: "tech"
description: "VimConf 2024参加&簡易レポです"
publish: true
thumbnail: "https://res.cloudinary.com/dkerzyk09/image/upload/v1732408198/blog/vim-conf-2024/jib55eh765wdihuspot2.png"
tags:
  - "tech/neovim"
  - "tech/イベント参加"
  - "tech/開発環境"
---

## はじめに

昨日(2024/11/23)[VimConf 2024](https://vimconf.org/2024/)に参加しました！

セッションは非常に充実した内容が多く、(Neo)Vimへのモチベーションが大きく上がる良い機会でした。

簡単ですが各セッションの感想を書きます！

:::message warning
個人の解釈を含みます。誤っている場合、以下までご連絡ください 🙇

https://github.com/shuntaka-dev/article/issues
:::

## セッションの感想

### Keynote - The new Vim project - What has changed after Bram

Christian Brabandtさんによる、Bram以降のVimのメンテナンスについての話。Bramに不幸があってから、Vimに関わる多くのメンテナ作業を引き継ぐ必要がありました。Vim本体だけでなく、例えばホームページのリプレースやメーリングリストの運用方法など多岐に渡り、多くの知識はBramのなかにあり仕事量も凄まじく、切実に感じられました。

例えば以下のような業務があります（セッションではより多くの業務について話がありました）：

* ホームページの移行
* FTPサーバーの停止
* ICCFへの寄付周り（Bram個人のチャリティ活動）
  * BramはICCFのチャリティの仕事は有名で、引き継いでも続けて行きたいとのこと
* メーリングリストのメンテナンス
* Security Reporting

Vim本体については以下のような話がありました：

* コントリビューションの敷居を下げる
* 開発者向けの取り組み
  * 安全なCコード
  * コードカバレッジ
  * 長いコードのリファクタ
  * python2、TCL、mzschemのようなインターフェースは必要に応じて引退させる
  * treesitterとの統合（近いうちに使えるようになるとのこと）

Vimのメンテナンスはハードで、フルタイムの仕事のようなもの、毎日取り組まないといけないことがあるという表現もありました。またコーディング作業だけでなく健全なコミュニティの維持やコミュニティの期待に答える必要があります。

月並みですが、Vimほどの規模のOSSとBramの意思も受け継いで膨大なメンテナ作業があり、維持されていることが分かる良い機会となりました。

### Keynote - (Neo)Vim Made Me a Better Software Developer

TJ DeVriesさんによる、Vimを通して良い開発者になれたという話。業務（電子カルテのシステム開発）では、自分との結びつきが遠い業界で、顧客理解の難易度が高く、システム開発が難しかったとのこと。医療に関わる基幹システムのため、開発フローは慎重だったそうです。

以上の業務と比較しNeovimのプラグイン開発は、顧客が自分であり、自分にとって大事でもあるため、モチベーションが維持しやすい環境でした。後方互換性もそこまで気にしなくて良く、全く新しいことが試せました。

業務でAPI設計がうまいと言われたのは、Neovimを通した試行錯誤の結果だったそうです。デザインパターンの勉強でそこまで到達はできず、複数のプラグイン開発を通して、失敗の連続の成果とのことでした。

良い開発者になるには、コーディングの遊び場を見つけることが大切です。自分にとってはそれがNeovimだったとのこと。何度も何度もアイディアを練習する場（playground）を見つけ、自分で開発し、自分が顧客となり、試行錯誤することが良い開発者になる道筋だと説明されました。

TJさんは[telescope.nvim](https://github.com/nvim-telescope/telescope.nvim)の作者であり、私自身使わない日はないくらい依存しているので、本セッションを非常に楽しみにしていました。

telescope.nvimの作者のキャリアの話には親近感を感じました。B2B向けのサービスではどうしても顧客目線になるのは難しいものです。自分の遊び場を持つという話は、自分も小さいツールを作ったり、ブログの自作を通じて多くのことを学んだので共感がありました。

### Mastering Quickfix

[📄](https://speakerdeck.com/daisuzu/mastering-quickfix)

Quickfixを使いこなせていなかったので非常に参考になりました。QuickFixとLocation listの機能の違いも理解していませんでした。スライドを読みながら手元で動作を確認して身に染み込ませたいと感じました！

### Hacking Vim script

[📄](https://docs.google.com/presentation/d/15QvYTshQ7n7S4MbQUSUN7aHB_d4P1hLlO9E-1-GVC6Y/edit#slide=id.g305584dd20c_1_0)

Vim Script自体に手を入れたり、ソースコードを追う話でした。

* Vim本体のコードに関数を追加し、デバッグする流れやソースコードの説明
* 膨大なソースコードの意図を知るためのgitでの変更追跡方法
* GC周りの説明
  * 参照カウント方式でGC。ただし到達不可能なオブジェクトは解放されないため、検査しているとのこと（PythonのGCを参考にしているなど）

Cのマクロが多用されていて、LSPが使えないのでctagsを使うなど、Vimのソースコードに立ち向かうためのTipsが非常に参考になりました。かなりディープな内容でしたが、説明が分かりやすかったです！Vim本体に手を入れる際には読み返したいと感じました！

### Switch between projects like a Ninja

session機能についての話でした。リポジトリ毎にセッションを管理して、高速にプロジェクトを切り替える手法が紹介されていました。git worktreeを使ったTipsもあり、大変参考になりました。

自分はtmuxで複数セッション管理していて、セッション数が爆発しやすいので早速導入したいと感じました。

### Vim meets Local LLM: Edit Text beyond the Speed of Thought

Local LLMを使ったソースコード補完のプラグインを作るという内容でした。Local LLMは重そうというイメージがありましたが、M1で動作可能とのこと。LLM Ollamaを使い、FIM（Fill in the Middle）に対応したモデルを使うことでソースコード補完を実現します。またVimプラグインでcopilot.vimのようなグレーで補完が先行で表示される体験をどう作るかについても参考になりました。

これからAI系のプラグインが増えてくると思うので、時代に合ったセッションでした！生成AIを少しかじっていたのでFIMという手法があることが知れてよかったです！

### Creating the Vim Version of VSCode Dev Container Extension: Why and How

[devcontainer.vim](https://github.com/mikoto2000/devcontainer.vim)のVimとDev Container連携拡張の話でした。VSCode拡張の技術的な説明が図付きで分かりやすかったです。コンテナ側にVS Code Serverを入れるようで、VSCodeのSSHと似ている印象を受けました。VSCode拡張を参考にdevcontainers cliを使ったり、連携時にdocker cpでVimを転送したり、一部ファイル（ソースコード、vimrc、gitconfig、ssh）はバインドマウントを使い機能を実現しているとのことです。チーム開発でVSCodeの設定との共存（devcontainer.vim.jsonと分ける）も考慮が必要なので、うまくケアされていて良いと感じました！

チームでdevcontainerの話になると自分はVimユーザーなので会話に入らないスタンスでしたが、今後は会話に参加できそうです！

### Neovim for Frontend Developers: Boosting Productivity and Creativity

[📄](https://talks.ryoppippi.com/2024-11-23/1)

フロントエンド開発におけるプラグインの紹介でした。自分の知らないプラグインが多くあり、全て試してみたいと感じました！主な紹介プラグインは以下の通りです：

* [ugaterm.nvim](https://github.com/uga-rosa/ugaterm.nvim)
* [nvim-highlight-colors](https://github.com/brenoprata10/nvim-highlight-colors)
* [mini.surround](https://github.com/echasnovski/mini.surround)
* [nvim-insx](https://github.com/hrsh7th/nvim-insx)
* [dial.nvim](https://github.com/monaqa/dial.nvim)
* [codecompanion.nvim](https://github.com/olimorris/codecompanion.nvim)
* [neotest.vim](https://github.com/nvim-neotest/neotest)
* [oil.nvim](https://github.com/stevearc/oil.nvim)
* [vim-sonictemplate](https://github.com/mattn/vim-sonictemplate)
* [vim-svelte-inspector](https://github.com/ryoppippi/vim-svelte-inspector)

longcatの演出で爆笑しました。テンポも良く、非常に面白かったです。

### Building Neovim Plugins: A Journey from Novice to Pro

Neovimプラグイン開発のTipsについての発表でした。

Neovim Luaでプラグイン開発がしたくなるセッションでした。プラグイン開発支援プラグインも多くあり、開発したくなったとき見返したい内容でした。

メモしたプラグインは以下の通りで、試したいみたい(N回目)

* [2KAbhishek/template.nvim](https://github.com/2KAbhishek/template.nvim)
* [plenary.nvim](https://github.com/nvim-lua/plenary.nvim)
* [nui.nvim](https://github.com/MunifTanjim/nui.nvim)
* [lazydev.nvim](https://github.com/folke/lazydev.nvim)
* [utils.nvim](https://github.com/2KAbhishek/utils.nvim)
* [nvim-luapad](https://github.com/rafcamlet/nvim-luapad)

### Can't Help Falling in Vim ~ Wise men say only fools reinvent the wheel, but I can't help building yet another fuzzy finder: Fall

Fuzzy Finderの歴史や新しいFuzzy Finder [fall.vim](https://github.com/vim-fall/fall.vim)の紹介でした。

Fuzzy finderプラグインの歴史の紹介があり、その変遷が感慨深かったです：

* FuzzyFinder
* unite.vim
* ctrlp.nvim
* fzf.vim（2015） Goの外部コマンドとしての実装（fzf内に実装があった）
* denite.nvim（2016）
* fzf-preview.nvim（2018）
* telescope.nvim（2020）
* ddu.vim（2021）
* Fall（2024）

挙動を忘れないようにするためのmapping helpやaction selector機能、サブマッチ機能など試したいと感じました。Latency over throughput（スループットよりレイテンシ）で体験を落とさないような思想もプロの仕事と感じました、、

Denoエコシステムを使い、カスタマイズがTS moduleでNPMやJSRで配信できるのは新しい試みで、TSばかり書いている私には大変ありがたく感じました！

### The latest dark deno powered plugins

Vim/Neovimエコシステムを支えているShougoさんのプラグインの設計思想についての発表でした。長年培った知識から作られた思想で、シャープでありながら洗練されていて、時にユニークで、会場が盛り上がったセッションでした。

設定させて頂きありがとうございます、、(ネタ抜きで)

### Lightning Talks（5min × 6）

こちらは簡単に、、申し訳ありません 🙇

* キャリアの変遷でVimの立ち位置は確かに変わりそうー
* iOSアプリかけるのは熱いなー！xcodebuild.nvim、nvim-dap、nvim-dap-ui覚えておきたい！
* Nix流行っているし、試してみたいー
* :helpコマンドの奥深い！
* テスト手法の紹介。プラグイン開発時参考にします！

## さいごに

お昼はすきやき弁当でとても美味しかったです！
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1732409699/blog/vim-conf-2024/uqaynnlvyeirg7zmbbbm.jpg)


懇談会は初参加でしたが、プロの(Vim/Neovim)merとVimだけでなくシェル、Go、Nix、LSP、ターミナルツールの話ができ幸せでした！話しくださった方ありがとうございました！

参加者の方、コンテキストが多い中での同時通訳の方、そして運営の方このような機会を設けて頂きありがとうございました✨

来年もまた参加したいなー✨


