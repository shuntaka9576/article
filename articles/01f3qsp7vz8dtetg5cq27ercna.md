---
title: "next/linkのpassHref属性がどのような場合に必要か整理、検証してみる"
type: "tech"
description: "next/link(Link属性、Linkコンポーネント)でpassHref属性の使い方を調査、検証しました"
publish: true
tags:
  - "tech/next.js"
  - "tech/react"
  - "tech/typescript"
---

# 結論
Linkコンポーネントのhref属性が、その子コンポーネントであるaタグに付与されないケースがあり、SEO的に悪影響なのでpassHref属性を使う必要がある。

* 子が **<a>タグをラップするカスタムコンポーネント**の場合、**passHrefすると**とaタグにhref属性を付与
* 子が **(<a>タグをラップした)関数コンポーネント**の場合、**passHrefかつReact.forwardRefでラップする**ととhref属性を付与

逆に上記のお作法を取らないと、href属性はつかない。詳しくは、項目[検証](#検証)で各パターン検証した。

# 公式ドキュメントを確認
公式ドキュメント[next/link](https://nextjs.org/docs/api-reference/next/link)の章題は下記。

1. If the route has dynamic segments(ルートに動的セグメントがある場合)
1. If the child is a custom component that wraps an <a> tag(子が<a>タグをラップするカスタムコンポーネントである場合)
1. If the child is a function component(子が関数コンポーネントの場合)
1. With URL Object(URLオブジェクト付き)
1. Replace the URL instead of push(プッシュの代わりにURLを置き換える)
1. Disable scrolling to the top of the page(ページ上部へのスクロールを無効にする)

`passHref`について言及しているのは、`2. If the child is a custom component that wraps an <a> tag`と`3. If the child is a function component`


# 検証
検証結果は下記。◯はaタグにhref属性が**付与されること**を表す。
|項|Linkコンポーネントでラップされたコンポーネント|Linkコンポーネント(passHref無)|Linkコンポーネント(passHref有)|
|---|---|---|---|
|1|aタグ|◯|◯|
|2|aタグをWarpしたカスタムコンポーネント||◯|
|3|FunctionalComponent|||
|4|React.forwardRefでWrapしたFunctionalComponent||◯|


検証コードは下記。コメントアウトでHTML出力結果も示します。
```ts
import React from 'react';
import Link from 'next/link';
import styled from 'styled-components';

/* aタグをWrapしたカスタムコンポーネント */
const CustomComponent = styled.a`
  color: red;
`;

/* FunctionalComponent */
const FunctionalComponent: React.VFC<{ value: string }> = ({ value }) => (
  <a>{value}</a>
);

/* React.forwardRefでラップしたFunctionalComponent */
type AncoherPorps = JSX.IntrinsicElements['a'];
type FunctionalComponentForwardRefProps = { value: string };
const FunctionalComponentForwardRef = React.forwardRef<
  HTMLAnchorElement,
  AncoherPorps & FunctionalComponentForwardRefProps
>(({ value, href }, ref: React.LegacyRef<HTMLAnchorElement>) => {
  return (
    <a href={href} ref={ref}>
      {value}
    </a>
  );
});

FunctionalComponentForwardRef.displayName = 'ForwardRefComponent';

const HogePage: React.VFC = () => {
  return (
    <div>
      {/* ----------- 表 項1 ----------- */}
      <Link href="/test">
        <a>testLink</a>
      </Link>
      <Link href="/test" passHref>
        <a>testLink</a>
      </Link>
      {/* HTML */}
      {/* <a href="/test">testLink</a> */}
      {/* <a href="/test">testLink</a> */}

      {/* ----------- 表 項2 ----------- */}
      <Link href="/test">
        <CustomComponent>{'testLink'}</CustomComponent>
      </Link>
      <Link href="/test" passHref>
        <CustomComponent>{'testLink'}</CustomComponent>
      </Link>
      {/* HTML */}
      {/* <a class="sc-bdvvaa hAZGnr">testLink</a> */}
      {/* <a href="/test">testLink</a> */}

      {/* ----------- 表 項3 ----------- */}
      <Link href="/test">
        <FunctionalComponent value={`testLink`} />
      </Link>
      <Link href="/test" passHref>
        <FunctionalComponent value={`testLink`} />
      </Link>
      {/* HTML */}
      {/* <a>testLink</a> */}
      {/* <a>testLink</a> */}

      {/* ----------- 表 項4 ----------- */}
      <Link href="/test">
        <FunctionalComponentForwardRef value={`testLink`} />
      </Link>
      <Link href="/test" passHref>
        <FunctionalComponentForwardRef value={`testLink`} />
      </Link>
      {/* HTML */}
      {/* <a>testLink</a> */}
      {/* <a href="/test">testLink</a> */}
    </div>
  );
};

export default HogePage;
```


# おまけ
React.forwardRefでラップする書き方、アローを使わないで書く方法は以下。

```ts
type AncoherPorps = JSX.IntrinsicElements['a'];
type FunctionalComponentForwardRefProps = { value: string };
const FunctionalComponentForwardRef = React.forwardRef<
  HTMLAnchorElement,
  AncoherPorps & FunctionalComponentForwardRefProps
>(function functionalComponentForwardRef(
  props,
  ref: React.LegacyRef<HTMLAnchorElement>,
) {
  return (
    <a href={props.href} ref={ref}>
      {props.value}
    </a>
  );
});
```
