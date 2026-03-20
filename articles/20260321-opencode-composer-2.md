---
title: "OpenCodeでComposer 2をCLIで試す"
type: "tech"
category: []
description: "小ネタです"
publish: true
---

## はじめに

2026年3月19日、Anysphere(Cursor開発元)がコーディング特化モデル「Composer 2」をリリースしました。

https://x.com/cursor_ai/status/2034668943676244133

Composer 2は中国発のオープンソースモデルKimi K2.5をベースに、コーディングデータセットでの継続事前学習と長期コーディングタスクでの強化学習を行ったモデルです。CursorBenchでは61.3を記録し、Claude Opus 4.6(58.2)を上回るスコアを出しています。また、価格面ではStandardモデルが入力$0.50/出力$2.50(per 1M tokens)と、Claude Opus 4.6の約1/10のコストで利用できます。

OpenCodeからComposer 2を利用するにはCursorのサブスクリプションが必要です。なおCursorのACP（Agent Client Protocol）対応と同様に、サブスク利用者がJetBrainsやNeovimなど好みのUIから利用することはCursor側も容認しています。

https://x.com/leerob/status/2034788399249244604

本記事では、OpenCodeの開発者であるEphraim Duncan氏が公開しているプラグイン[ephraimduncan/opencode-cursor - GitHub](https://github.com/ephraimduncan/opencode-cursor)を使って、OpenCodeからComposer 2を利用する手順と所感をまとめます。

https://x.com/ephraimduncan/status/2034753448768335997

## 連携設定

前述のプラグインを設定ファイルに記述します。

```diff:~/.config/opencode/opencode.jsonc
diff --git a/home-manager/programs/opencode/opencode.jsonc b/home-manager/programs/opencode/opencode.jsonc
index 07a3e05..b470d18 100644
--- a/home-manager/programs/opencode/opencode.jsonc
+++ b/home-manager/programs/opencode/opencode.jsonc
@@ -3,5 +3,11 @@
   "theme": "opencode",
   "model": "glm-4.7-free",
   "autoupdate": false,
+  "plugin": ["opencode-cursor-oauth"],
+  "provider": {
+    "cursor": {
+      "name": "Cursor"
+    }
+  },
   "mcp": {}
 }
```


以下のコマンドを実行し、cursor.comのURLをコピーしブラウザに入力します。

```bash
$ opencode auth login --provider cursor

┌  Add credential
│
●  Go to: https://cursor.com/loginDeepControl?challenge=[challenge]&uuid=[uuid]&mode=login&redirectTarget=cli
│
●  Complete login in your browser. This window will close automatically.
│
◐  Waiting for authorization.
```

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1774039155/blog/2026-03-21-opencode-composer-2/nlfzabzxduxw5v1u7kwa.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1774039433/blog/2026-03-21-opencode-composer-2/ixltl5phde463ygvxejp.png)


認証が完了していることを確認します。
```bash
$ opencode auth login --provider cursor


┌  Add credential
│
●  Go to: https://cursor.com/loginDeepControl?challenge=[challenge]&uuid=[uuid]&mode=login&redirectTarget=cli
│
●  Complete login in your browser. This window will close automatically.
│
◇  Login successful
│
└  Done
```

普通にOpenCodeを起動します。

```bash
$ opencode
```

`/models` で `Composer` で閲覧可能なことを確認します。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1774039544/blog/2026-03-21-opencode-composer-2/opskhf0zwx6pku5tncsr.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1774039639/blog/2026-03-21-opencode-composer-2/kr7tjua3c7fbfralnefy.png)


アプリ2000行程度コードベースに対して実行してみます。速度は等倍です。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1774040623/blog/2026-03-21-opencode-composer-2/phhutvw0oxyxr4limo1n.gif)

上記はplan作成ですが、コード生成やコマンドラインツール実行→フィードバックループはサクサクな感じです🧐 しばらく使ってみようと思います！

`/sessions` から履歴を復元した場合、内容忘れている気がする...🥺 これは別途調査してみます。。画像は1で `/new` して 2で `/sessions` したところ忘れているように見える例。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1774041600/blog/2026-03-21-opencode-composer-2/rwlnfgu1nxn72barpkki.png)

## 参考

https://cursor.com/blog/composer-2

