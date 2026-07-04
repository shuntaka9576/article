---
title: "2020年の振り返り"
type: "tech"
description: "2020年も残すところあと2日となりました！！お世話になった皆様、感謝です！本記事では、2020年の個人活動を軽く振り返ります。"
thumbnail: "https://res.cloudinary.com/dkerzyk09/image/upload/v1612594581/blog/01etqfnfw9h98gffzbqsv4r32w/daflgl5t40jdtiam4khv.webp"
publish: true
tags:
  - "misc/振り返り"
  - "tech/neovim"
  - "tech/next.js"
  - "tech/go"
---

# 作ったツール/サイト
## preview-asciidoc.nvim
リポジトリ[こちら](https://github.com/shuntaka9576/preview-asciidoc.nvim)。NeovimでAsciidocをリアルタイムプレビューするプラグイン。業務でAsciidocの中でPluntUMLを使うことが多く、VSCodeのリアルタイムプレビュープラグイン(joaompinto.asciidoctor-vscode)の場合、1ファイルに20個くらいシーケンスがあると、メモリが足りなくなることがあったのと普段Neovimを使うので作りました。

メモリを使い過ぎていた理由は、エディタでカーソルを動かす度にPluntUMLの画像生成コマンドが大量に発行されているからかなーと思い、コマンド処理中はフラグを立ててフラグがある間はコマンドを発行しないようにしたら、割といい感じに使えるプラグインになった。

同じプロジェクトの同僚にもさくっと導入してもらい、好評で良かった。

詳細は[NeovimでSwaggerをリアルタイムプレビューするプラグインを作った話](https://qiita.com/hozi894/items/d9b296f923b2d394a26f)。

## preview-swagger.nvim
リポジトリは[こちら](https://github.com/shuntaka9576/preview-swagger.nvim)。NeovimでSwaggerをリアルタイムプレビューするプラグイン。前項と同じ要領で作りました。Neovimでプラグインを作るのは2020年の目標で達成できて良かった。

## kanban
リポジトリは[こちら](https://github.com/shuntaka9576/kanban)。GitHub ProjectのTUI。正直あんまり活躍してない..。GitHub Project自体のUXがいいというのが原因..。

UIをブロックしない方がUXがよく、自然とgoroutineを使った実装へ強制される。Goを使っているがgoroutine使う機会がない人はブロックしないTUIを作ってみるのが良い学習材料になるかも(?)。

詳細は[GitHubプロジェクト(カンバン)をターミナルで確認するツールを作ってみた](https://qiita.com/hozi894/items/a3cbb36c569fae649e77)。

## 自ブログ
本ブログをフルスクラッチで作った。Zennに影響されて、Next.js(Vercel)でISRするブログを作った。ISR時には、サーバレスAPI(TS × Lambda × DynamoDB)を呼び出している。MarkdownItで変換されたHTMLにCSSで装飾して自前のSSGを作るのは結構楽しかった(粉ミカン)。

フロント側は経験が浅くゴリ押ししてる箇所が多いので、2021年はいい感じに育てたい。

# 業務
業務ではAWSでサーバレスバックエンド、ときどきIoTという感じでした。アウトプットが少ないので、来年こそ..(1年ぶり2度目)。

# 2021年向けて
* shuntaka.devを実験場にして、色々味見する
  * 認証を入れて、スマホからでも簡単な編集を出来るようにする
  * BEMやCSS Moduleは雰囲気理解なので、手に馴染ませたい
  * algoriaやStripeの導入(何に使うんだ!)
  * 技術ネタとはサイトのパスを分けて、ガジェットの記事を書きたい。界隈はWordPressだし、差別化できそう(?)
* メンターをやることが多いので、自己流はやめて書籍や事例から効果的なやり方を取り入れる
* 純粋にアウトプットを増やす(週1目安)

# 最後に
良いお年を〜

# その他アウトプット
* [GitHubを操作するCLI「gh – The GitHub CLI tool」(Beta)を試す](https://dev.classmethod.jp/articles/shuntaka9576-gh/)
* [AWS IoT ルールで配列操作が安全に出来ず、正規表現で解決した話](https://dev.classmethod.jp/articles/shuntaka-aws-iot-regex-rule/)
* [ESLintでTypeScriptの変数の名付け規則をチェックしよう！](https://dev.classmethod.jp/articles/shuntaka9576-check-eslint/)
* [CDKでAWS Lambdaのパッケージフォーマットにコンテナイメージを指定してデプロイしてみた](https://dev.classmethod.jp/articles/shuntaka-container-image-support-lambda-cdk/)

