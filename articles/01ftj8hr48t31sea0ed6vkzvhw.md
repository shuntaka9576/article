---
title: "tailwindcss入門(随時更新)"
type: "note"
description: "tailwindcss入門"
thumbnail: ""
publish: true
tags:
  - "tech/tailwindcss"
  - "tech/css"
  - "tech/react"
---

:::message warning
[tailwindcss公式](https://tailwindcss.com/docs)の雑メモ
:::


# [Padding](https://tailwindcss.com/docs/padding)
|class名|説明|
|---|---|
|p(t\|r\|b\|l)-{数字}|t(上), r(右), b(下), l(左)方向のpadding
|px-{数字}|左右のpadding
|py-{数字}|上下のpadding
|p-{数字}|上下左右のpadding
|hover:py-8|hoverしたとき`py-8`を適用


# [FontSize](https://tailwindcss.com/docs/font-size)

|class名|font-size|line-height|
|---|---|---|
|text-xs|0.75rem|1rem|
|text-sm|0.875.rem|1.25rem|
|text-base|1rem|1.5rem|
|text-lg|1.125rem|1.75rem|
|text-xl|1.25rem|1.75rem|
|text-2xl|1.5rem|2.rem
|text-3xl|1.875rem|2.25rem
|text-4xl|2.25rem|2.5rem
|text-5xl|3rem|1
|text-6xl|3.75rem|1
|text-7xl|4.5rem|1
|text-8xl|6rem|1
|text-9xl|8rem|1


# [Font Weight](https://tailwindcss.com/docs/font-weight)

|class名|font-weight|
|---|---|
|font-thin|100|
|font-extralight|200|
|font-light|300|
|font-normal|400|
|font-medium|500|
|font-semibold|600|
|font-bold|700|
|font-extrabold|800|
|font-black|900|

# [Margin](https://tailwindcss.com/docs/padding)

|class名|説明|
|---|---|
|m(t\|r\|b\|l)-{数字}|t(上), r(右), b(下), l(左)方向のmargin
|mx-{数字}|左右のmargin
|my-{数字}|上下のmargin
|mp-{数字}|上下左右のmargin

# [Text Alignment](https://tailwindcss.com/docs/text-align)

|class名|text-align|
|---|---|
|text-left|left|
|text-center|center|
|text-right|right|
|text-justify|justify|


# [Algin Items](https://tailwindcss.com/docs/align-items)

|class名|Properties|
|---|---|---|
|items-start| align-items:flex-start;|配置コンテナの先頭側の端に寄せられる
|items-end|align-items: flex-end;|配置コンテナの末尾側の端に寄せられる
|items-center|align-items: center;|フレックスアイテムのマージンボックスは、交差軸上の中央に配置される。アイテムのcross-sizeがフレックスコンテナより大きい場合は、両方向へ均等にはみ出す
|items-baseline|align-items: baseline;|
|items-stretch|align-items: stretch;|アイテムのマージンボックスのcross-sizeが、幅や高さの制約の範囲内でラインと同じになるように拡張される

```ts
<div class="flex items-stretch">
  <div class="py-4">01</div>
  <div class="py-12">02</div>
  <div class="py-8">02</div>
</div>
```

TODO: `items-start`は寄せられるとは書いてあるものの、実例ではその挙動がわからなかったので確認

# [Space Between](https://tailwindcss.com/docs/space)
小要素間のスペースを制御するためのユーティリティ

`space-{x|y}-{amount}`


```ts:横並びで間隔を調整する場合
<div calss="flex space-x-4"
  <div>01</div>
  <div>02</div>
  <div>03</div>
</div>
```

```ts:縦並びで間隔を調整する場合
<div calss="flex flex-col space-y-4"
  <div>01</div>
  <div>02</div>
  <div>03</div>
</div>
```

# [Text Color](https://tailwindcss.com/docs/text-color)
色の指定、省略

|class名|説明|
|---|---|
|text-{}|textの色指定|
|bg-{}|背景の色指定|
|ring-{}|アウトラインリングの色指定|

# [Border Radius](https://tailwindcss.com/docs/border-radius)
角の丸み指定、省略

* rounded-{}
  * rounded-none
  * rounded-sm
  * rounded-md
  * rounded
  * etc ...

# [Border Width](https://tailwindcss.com/docs/border-width)
枠線の太さ
* border-{}

## サンプル
border-b-2...下に2px
border-blue-500...2pxにblue-500の色付け

# [Top / Right / Bottom / Left](https://tailwindcss.com/docs/top-right-bottom-left)
位置決めされた要素の配置を制御する。リンク参照。形式は、`{top|right|bottom|left|inset}-0`の形。insetは、top,right,bottom,leftの一括指定を示す
項目リンクの画像をみつつ、クラスが推測できるようになるといいかもしれない

`left-0 top-0`とみたら、左から0の上から0の座標と読み替えると良い

```ts: 32に正方形の中に、16の正方形を左上に配置する場合
<div calss="relative h-32 w-32" ...>
  <div class="abusolute left-0 top-0 h-16 w-16">01</div>
</div>
```

```ts: 32に正方形の中に、縦=16横=32の正方形を上に配置する場合
// `inset-x-0`でright:0;left:0を指定

<div calss="relative h-32 w-32" ...>
  <div calss="absolute inset-x-0 top-0 h-16>"02</div>
</div>
```

```ts: コンテナを全て埋める
<div calss="relative h-32 w-32" ...>
  // ↓サイズ指定はいらいない(?)
  <div class="abusolute inset-0">05</div>
</div>
```


# [Ring Width](https://tailwindcss.com/docs/ring-width)
ボックスシャドウを使ったアウトラインリング

* ring-{}
  * ring-0
  * ring-1
  * ring-2
  * ring



# [Responsive Design](https://tailwindcss.com/docs/responsive-design)
`sm:`だと、このブレークポイントより上の..という意味を指す

|BreakPoint prefix|Minimum width|CSS
|---|---|--|
|sm|640px|
|md|768px|
|lg|1024px|
|xl|1280px|
|2xl|1536px|

以下の例の場合
* 接頭辞なしのtext-center -> モバイル適用
* sm:text-left -> smより大きい場合適用
```ts
<div class="text-center sm:text-left" />
```

上記のような接頭辞のないユーティリティをモバイルのターゲットにして、より大きなブレークポイントで上書きする書き方が良いみたい。


## Sample
接頭辞なしは、モバイル適用と意識しておくとレスポンシブなtailwindcssを読み解くことができそう

```ts
<div class="sm:hidden">
 // モバイルより大きい場合、hidden
<div>
<div class="hidden sm:block">
 // モバイルでdisplay:hidden
</div>
```


# [Handling Hover, Focus, and Other States](https://tailwindcss.com/docs/hover-focus-and-other-states)

hoverしたときに、bg-sky-700に状態変化する
```ts
<button class="bg-sky-600 hover:bg-sky-700">
  Save chages
</button>
```

## [Hover, focus, and active](https://tailwindcss.com/docs/hover-focus-and-other-states#hover-focus-and-active)

### 基礎的な前提

擬似クラスの確認

|オプション|説明|
|---|---|
|:link|リンクのデフォルトの状態
|:visted|既にクリックされているリンクの状態
|:hover|マウスをリンクの上に置いたときの状態
|:active|アクティブがされた状態
|:focus|タッチデバイスでタップした時や、キーボードのtabキーで移動した時


### サンプル

`outline-none`

```ts
<button class="bg-violet-500 hover:bg-violet-400 active:bg-violet-600 focus:outline-none focus:ring focus:ring-violet-300 ...">
  Save changes
</button>
```


要件
* hoverすると、少し暗くなる
* 押下すると枠線強調表示
```ts
<button
  type="button"
  className="py-2.5 px-5 mb-2 text-sm font-medium text-center text-white bg-purple-700 hover:bg-purple-800 dark:bg-purple-600 dark:hover:bg-purple-700 rounded-full focus:ring-4 focus:ring-purple-300 dark:focus:ring-purple-900"
>
  Purple
</button>
```

|オプション名|解説|
|---|---|
|py-2.5|上下padding2.5|
|px-5|左右padding5|
|mb-2|下margin2
|text-sm|font-size指定
|font-medium|font-weight指定
|text-center|text-alignを中心へ
|text-white|text色指定
|bg-purple-700|背景色を指定
|hover:bg-purple-800|ホバーしたら、背景色をpurple-800へ
|dark:bg-purple-600|ダークモードのときの背景色
|dark:hover:bg-purple-700|ダークモードかつホバーしたときの背景色
|rounded-full|border-radius: 9999px;指定
|focus:ring-4|アウトラインリング
|focus:ring-purple-300|アウトラインリングの色指定
|dark:focus:ring-purple-900|ダークモード のアウトラインリングの色指定



## [Dark Mode](https://tailwindcss.com/docs/dark-mode)

以下のように`dark`variantを利用する

```ts
dark:bg-slate-900
```


## FontFamily

```ts
module.exports = {
  content: ['./src/index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      fontFamily: {
        'sans': ['sans-serif'],
        'relics': ['Times New Roman', 'MS PGothic'],
        'ui': ['system-ui', '-apple-system']
      }
    }
  },
  plugins: [],
};
```

```ts
const defaultTheme = require("tailwindcss/defaultTheme");

module.exports = {
  content: ['./src/index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      fontFamily: {
        sans: ["Noto Sans JP", ...defaultTheme.fontFamily.sans],
      },
    }
  },
  plugins: [],
};
```

# [Position](https://tailwindcss.com/docs/position)

|クラス名|Properties|説明|
|static|position: static;|
|fixed|position: fixed;|固定
|absolute|position: absolute;|
|relative|position: relative;|
|sticky|position: sticky|


staticで基準の領域を示し、abusoluteで相対的なdivを配置する例
```ts
<div class="static ...">
  <p>Static parent</p>
  <div class="abusolute bottom-0 left-0 ...">
    <p>Abusolute child</p>
  </div>
  </div>
</div>J
```


# [Flex](https://tailwindcss.com/docs/flex)

|クラス名|Properties|説明|
|---|---|---|
|flex||要素横並びにフレックコンテナを作るクラス。flexコンテナ要素につけやつ
|flex-1|flex: 1 1 0%|フレックスアイテムは初期サイズを無視して、必要に応じて伸縮する
|flex-auto|flex: 1 1 auto)|flex-1より初期サイズを考慮して、flexコンテナが伸縮する
|flex-initial|flex: 0 1 auto)|flexコンテナに合うように伸縮する
|flex-none|flex: none,flex 0 0 auto|width, hight固定。指定されいる要素は伸縮しない| 

例
```ts
<div class="flex">
  <div calss="flex-none"
</div>
```

* [参考](https://developer.mozilla.org/ja/docs/Web/CSS/flex)



# [Pointer Events](https://tailwindcss.com/docs/pointer-events)

|class| Properties|説明
|---|---|---|
|pointer-events-none|pointer-events:none;|マウスイベント発生しない。小要素にpointer-events:autoが設定された場合、小要素のマウスイベントは発生
|pointer-events-auto|pointer-events:auto;|通常の動作


# [Transitions & Animation](https://tailwindcss.com/docs/transition-duration)

## [Transition duration](https://tailwindcss.com/docs/transition-duration)
`duration-{amount}`という記法。CSSの遷移の期間を制御するユーティリティ。大きいほど遅くなる。(例えばボタンをホバーしたらボタンが大きくなる。大きくなるまでの時間のことを指す。)


## [Transition Timing Function](https://tailwindcss.com/docs/transition-timing-function)

|クラス名|Properties|説明|
|---|---|---|
|ease-liner|transition-timing-function: linear;|等速(線形に動く)
|ease-in|transition-timing-function: cubic-bezier(0.4, 0, 1, 1);|動き出しゆっくり、最後に早く
|ease-out|transition-timing-function: cubic-bezier(0, 0, 0.2, 1);|動き出し高速、最後に減速
|ease-in-out|transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);|動き出し、最後がゆっくりで中盤は加速

[速度感のイメージをつかみたい場合は本リンク](https://developers.google.com/web/fundamentals/design-and-ux/animations/the-basics-of-easing?hl=ja)を参照

## 例

```ts: アニメーション加速度にease-in-out、遷移する長さに150を指定
<button class="transition duration-150 ease-in-out">A</button>
```


## transformとtransition
|プロパティ|解説
|---|---|
|transition|始点と終点のアニメーションを変化さenterせるプロパティ
|transform|x,y座標を利用 移動(translate)、回転(rotate)、拡大縮小(scale)などの動きをつけラエル|


# サンプル解説

## button

|観点|例
|---|---|
|テキストサイズ|text-sm
|フォントサイズ|font-medium
|テキストカラー|text-indigo-600
|テキストカラー(hover)|hover:text-indigo-500
|バックグラウンドカラー|bg-white
|ボタンの丸み|rounded-md
|フォーカスしたときの枠幅|focus:ring-2
|フォーカスしたときの枠色|focus:ring-indigo-500
|枠をボタンの外側にする場合|focus:ring-offset-2|
|Chromeなどで、テキストエリアのフォーカス時に自動付加するピンク色の枠を消す|focus:outline-none(※1)


※1 input type="search"でも、本オプションはデフォルトで利用するべき


## Header
以下のようなヘッダーを作る時

flex-colで以下のように全体的に領域を切る
```bash
[header]
[content]
[footer]
```


```bash
|Logo|SearchBox menu1 menu2 menu3|BellIcon|
```




# eslint-plugin-tailwindcss

## ソート規則

* サイズ、テーマ、状態の順に記載する


# 疑問点

* aria-live="assertive"の意味


