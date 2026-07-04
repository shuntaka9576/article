---
title: "Figma備忘録"
type: "note"
description: "Figmaの備忘録を随時更新していきます"
thumbnail: ""
publish: true
tags:
  - "tech/figma"
  - "tech/開発環境"
---

:::message warning
随時更新中
:::

# ショートカット

|ショートカット|説明|
|---|---|
|f|Frameを作成|
|t|textを作成。ボックスは作らない方がよい。文字のながさぴったりなるため。|
|cmd+マウス(Frameの伸縮)|内部の要素の伸縮させずFrameの長さを調整|
|cmd+option+G|選択要素をFrameに統合|
|cmd+option+k|選択要素をコンポーネント化|

# Auto Layout系

## Auto Layoutのオプション
### Hug contents
FrameとTextでButtonを作成した際に、中身のTextを増やしたら追従してFrameが拡張する

### Fixed width/hight
Frameが自動拡張子ない

## Resizeing
### 親の要素
### 子の要素
Auto Layoutが適用された子要素にもResizeingのオプションが設定可能

#### Fil container
親要素の長さに合わせる
```bash
-parent----------------
|-text-               |
||    |               |
||----|
-----------------------
↓ text要素(子要素)をFil container
-parent----------------
|-text-----------------|
||                    ||
||--------------------||
-----------------------
```

### 右寄せしたい場合

下記のような要素配置。要素1-3を統合している全体要素を拡張した際に要素3は、右寄せしたい。

```bash
|要素1| |要素2| |要素3|
```

下記の設定を追加
* 要素1と要素2がFrameに統合されてAutoLayout設定済 -> 要素4とする
* 要素4と要素3がFrameに統合されてAutoLayout設定済

デフォルトだと、要素4と要素3に横方向のAutoLayoutが効くため、右側に寄ってくれない
Auto Layout > Alinment and padding > PackedをSpace betweenに変更する

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1642837850/blog/01fqtbggce9a5g9efzf83rvydy/olh0aa4r4qdi6ipbcf12.png)

これはオプションが入れ子で結構わかりにくので注意
