---
title: "NeovimのUIコンポーネントライブラリnui.nvimに関するメモ"
type: "note"
description: ""
publish: false
---

# はじめに
nui.nvimは、NeovimのためのUI Componentライブラリです。自分はまだまだbufferやwindow,floating windowの仕様や取り回しについて知識がないため、まずライブラリから入門してみます。

# Popup
[README.md](https://github.com/MunifTanjim/nui.nvim/tree/main/lua/nui/popup)

```lua: サンプル
local popup = Popup({
  position = "50%",
  size = {width = window_width, height = 2},
  enter = true,
  focusable = true,
  zindex = 50,
  relative = "editor",
  border = {
    highlight = "FloatBorder",
    padding = {top = 0, bottom = 0, left = 0, right = 0},
    style = "rounded",
    text = {top = "DeepL translate"}
  },
  buf_options = {modifiable = true, readonly = true},
  win_options = {winblend = 10, winhighlight = "Normal:Normal"}
})
```

## コンフィグオプション

|オプション|説明|
|---|---|
|position|
|size|
|enter|表示されたバッファの中にカーソルが入る
|focusable|
|zindex|
|relative|`cursor`, `editor`, `win`
|border.highlight|
|border.padding|
|border.style||
|border.text||
|buf_options.modifiable|
|buf_options.readonly|
|win_options.winblend|
|win_options.winhighlight|

### relative
IDE等でよくある文字列の下にpopupを出したい場合は、`cursor`が良いと思う。私は始め`cursor`というオプションを知らず、`editor`に対して相対的な位置関係を自前で算出していた。

```lua
local win_id = vim.fn.win_getid()
-- Neovimのエディタ全体でのウィンドウの左上座標を算出
local win_screenpos = vim.fn.win_screenpos(win_id)
local win_screen_row = win_screenpos[1]
local win_screen_col = win_screenpos[2]

-- ウィンドウ中の座標を取得
-- 別ウィンドウで分割されている場合の考慮されない
local win_col = vim.fn.wincol()
local win_row = vim.fn.winline()

local popup_row = win_row + win_screen_row - 1
local popup_col = win_col + win_screen_col - 2

popup:set_position({row = popup_row, col = popup_col})
```


## 関数オプション

|Popup:関数名|説明|
|---|---|
|init(options)|Popupの初期化
|_open_window()|内部関数のため省略
|_close_window()|内部関数のため省略
|mount()|Popupの表示|
|hide()|
|show()|
|unmount()|Popupの削除
|map(mode, key, handler, opts, force)|Popupのキーバインド設定
|on(event, handler, options)|eventが発火した際のバッファの挙動を定義
|off(event)|
|set_size(size)|
|set_position(position, relative)|

### unmount
あまり使う機会は少ないかもしれないが、一定時間後にpopupを削除する場合は下記のようになる。


```lua
local timer = vim.loop.new_timer()
timer:start(1000, 0, vim.schedule_wrap(function() popup:unmount() end))
```


## フィールド

|フィールド名|説明|
|---|---|
|bufnr|


# メモ

## utf8周りで試した事柄

```lua
  local text = "aaa、テスト"
  -- for word in text:gmatch("[\33-\127\192-\255]+[\128-\191]*") do
  --   print(word)
  --   print("--")
  -- end
  print("nagasa")
  for i = 1, #text do
    -- print(string.sub(text, i, i))
    print(string.byte(string.sub(text, i, i)))
    print(vim.fn.nr2char(string.byte(string.sub(text, i, i))))
   -- print(vim.fn.nr2char(word))
  end

  local popup_height = math.ceil(string.len(translate_sentence) / popup_width)

  local text = "aaa、テスト"
  print("len" .. utf8.len(ja_200))
  print("len" .. utf8.len(en_200))
  print("len" .. string.len(ja_200))
  print("len" .. string.len(en_200))

  -- for i = 1, #ja_200 do
  --   print(string.byte(string.sub(ja_200, i, i)))
  --   -- print(ja_200[i])
  -- end

  local display_length = 0
  local preview_pos = nil
  for p, _ in utf8.codes(translate_sentence) do
    if preview_pos == nil then
      preview_pos = p
      display_length = display_length + 2
    else
      local size = p - preview_pos
      if size == 3 then
        display_length = display_length + 3
      else
        display_length = display_length + 1
      end
    end
  end

  print(display_length)
  popup_height = math.ceil(display_length / popup_width)
  -- local popup_height = calc_translate_popup_popup_hight(win_hight)
  local utf8 = require("utf8")
```

## nvim_buf系についての調査する

```lua
  print("bufnr:" .. tostring(popup.bufnr) .. " nvim_buf_is_valid:" ..
          tostring(vim.api.nvim_buf_is_valid(popup.bufnr)) ..
          " nvim_buf_is_loaded:" ..
          tostring(vim.api.nvim_buf_is_loaded(popup.bufnr)))
```

## plenary.curl
```lua
  curl["post"]({
    -- url = "https://reqres.in/api/users?page=5",
    -- TODO: ある程度不可をかけたい場合
    -- url = "https://api.hozi.dev/v1/users/shuntaka/articles?type=tech",
    -- TODO: 本番
    url = "https://api-free.deepl.com/v2/translate",
    accept = "application/json",
    body = {auth_key = token, text = selected_lines, target_lang = config.to},
    callback = vim.schedule_wrap(call_api_and_display_popup)
  })
```
