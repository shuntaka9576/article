---
title: "Headless UIのオプション調査"
description: "Headless UIのオプション調査"
thumbnail: ""
publish: true
tags:
  - "tech/react"
  - "tech/tailwindcss"
---

:::message warning
[Headless UI](https://headlessui.dev/react)の雑メモ
:::

# [Transtion](https://headlessui.dev/react/transition)

|オプション|解説|
|---|---|
|enter|要素が入力されている間、ずっと適用される。通常、ここで期間と遷移させたいプロパティを定義します。例えば`transition-opacity` `duration-75`|
|enterForm|入力開始時点。フェードインの場合 `opacity-0`(透明)
|enterTo|入力終了時点。フェードインの場合 `opacity-100`(不透明)
|leave|要素が離脱している間に適用。ここで期間と遷移させたいプロパティを定義します。例えば`transition-opacity` `duration-75`
|leaveFrom|離脱の開始時点。フェードアウトの場合は、`opacity-100`(不透明)
|leaveTo|離脱の終了時点。フェードアウトの場合は、`opacity-0`(透明)

ポイントshowプロパティ。

1. 初期状態isShowing(true)
2. ボタンを押下
  2.1. isShowingがtrue -> false
  2.2. leave,leaveFrom(不透明) -> leaveTo(透明)
  2.3. 500ms待機
  2.4. isShowingがfalse->true
  2.5. enterFromの状態(透明+120度傾き+サイズ50)
    2.5.1. 120度傾きから0になるように回転
  2.6. enterToの状態(不透明+0度傾き+サイズ50)へ、enterで指定した間隔(400ms)で遷移する



```ts:サンプル
const [isShowing, setIsShowing] = useState(true);
const [, , resetIsShowing] = useTimeoutFn(() => setIsShowing(true), 500);

return (
  <div>
    <MainEcosystem>
      <div>
        <div className="flex flex-col bg-black items-center py-16">
          <div className="w-32 h-32">
            <Transition
              as={Fragment}
              show={isShowing}
              enter="transform transtion duration-[400ms]"
              enterFrom="opactiy-0 rotate-[-120deg] scale-50"
              enterTo="opactiy-100 rotate-0 scale-100"
              leave="transform duration-200 transtion ease-in-out"
              leaveFrom="opactiy-100 rotate-0 scale-100"
              leaveTo="opactiy-0 scale-95"
            >
              <div className="w-full h-full bg-white rounded-md shadow-lg" />
            </Transition>
          </div>
        </div>
        <button
          onClick={() => {
            setIsShowing(false);
            resetIsShowing();
          }}
          className="flex items-center px-3 py-2 mt-8 text-sm font-medium text-white transtion transform bg-black rounded-full backface-visibility-hidden active:bg-opacity-40 hover:scale-105 hover:bg-opacity-30 focus:outline-none bg-opacity-20"
        >
          <span className="ml-3">Click to transtion</span>
        </button>
      </div>
    </MainEcosystem>
  </div>
);
```
