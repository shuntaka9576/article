---
title: "Apple Developer Programへようこそ(¥12800)"
description: "57時間57分の物語"
publish: true
tags:
  - "tech/開発環境"
  - "tech/macos"
  - "tech/tauri"
---

## はじめに

最近macOSのAppを作りたくなった。Tauriでアプリを作成した。ただmacOSはApple Developer未課金アプリを起動するとゴミ箱へプロキシされる。実際危ないので仕方ない。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1771107043/blog/2026-02-15-apple-developer/x4ivq7gtphvp7wfzwg6o.png)

macOSはインターネットからダウンロードしたアプリに com.apple.quarantine 属性を付与する。Gatekeeperは起動時にこの属性を検出し、Appleの署名 + 公証がないアプリをブロックする。

ワークアラウンドはあって、以下のように `com.apple.quarantine` 属性を消せば起動出来る。

```bash
xattr -cr /Applications/Agentoast.app
```

ダウンロードしたアプリに付与されるのでもちろんGitHub Releaseからdmgを落としてもcaskからcurl経由でも同じ。

並行でネイティブアプリを作っていることもあり、開発者プログラムに登録することにした。

## 開発者プログラム登録

Apple Developerのアプリをインストールする。

https://apps.apple.com/jp/app/apple-developer/id640199958

今すぐ登録から開始する。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1771106126/blog/2026-02-15-apple-developer/m8zudp9pekuykuimwck2.png)

この先はマスクミスるとあれなので、スクショは割愛する 🙇 以下は全部日本語で書いた。結論問題なかった。

1. 運転免許かパスポートを用意してと言われる
2. カメラが起動して、1で選択した書類の写真をとる(カメラの切り替えボタンがあるので外部カメラの人はそれで対応可能)
3. 次に住所氏名を入れるフォームに遷移, 入力
4. 同意して、Appleのいつもの課金画面が表示され課金

1,2で自分は運転免許で3に進まず、パスポートにしたらすんなり3に遷移した。2でうまく撮れてなくてバリデートエラーだったのかな🧐

そんなこんなで日曜日のJST AM 6時40分に課金が完了して以下の画面になった。ちなみにJST AM 7時は米国西海岸(PST)だと土曜の14時にあたる。米国は週末のため、承認は早くても翌週になりそう。Apple側で承認されるまで証明書は作れないのでお預けという感じ。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1771333564/blog/2026-02-15-apple-developer/csvxm6b4fywv6flrwhsl.png)

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1771105973/blog/2026-02-15-apple-developer/v3d3x24a9loe0f4wjebu.png)

火曜日の16時37分に承認メールが来ました。日曜 AM 6:40の課金完了から2日と9時間57分（約57時間57分）で、週末を挟んでいることを考えると実質1〜2営業日で承認された！

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1771333385/blog/2026-02-15-apple-developer/yhuioohchhbgegeeduvo.png)

承認は最大48時間みたいです！Appleさん課金させて戴きありがとうございます！！
