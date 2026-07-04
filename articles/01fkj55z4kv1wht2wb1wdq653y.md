---
title: "Programming in Luaをやってみた際に詰まったところ"
type: "tech"
description: ""
publish: false
---

# 14章 環境

## グローバル変数の出力

```lua:グローバル変数一覧の出力
local function getfiled(f)
  local v = _G
  for w in string.gmatch(f, "[%w_]+") do
    v = v[w]
  end
  return v
end

for n in pairs(_G) do
  local value = getfiled(n)
  io.write(n .. ":")
  print(value)
end
```


:::deatils 出力結果
```bash
math:table: 0x7fc2dfc08b50
dofile:function: 0x108149290
package:table: 0x7fc2dfc06810
arg:table: 0x7fc2dfc096a0
getmetatable:function: 0x10814936d
tostring:function: 0x108149ca9
rawequal:function: 0x108149807
setmetatable:function: 0x1081499e8
_G:table: 0x7fc2dfc05ef0
require:function: 0x7fc2dfc06eb0
rawget:function: 0x1081498a4
_VERSION:Lua 5.4
rawset:function: 0x1081498ef
assert:function: 0x10814903d
print:function: 0x10814969d
table:table: 0x7fc2dfc07720
error:function: 0x1081492fc
load:function: 0x108149471
collectgarbage:function: 0x1081490b8
debug:table: 0x7fc2dfc076c0
string:table: 0x7fc2dfc08420
os:table: 0x7fc2dfc073f0
warn:function: 0x108149762
type:function: 0x108149cd7
utf8:table: 0x7fc2dfc06b70
tonumber:function: 0x108149a81
rawlen:function: 0x10814984f
next:function: 0x10814955f
io:table: 0x7fc2dfc07920
coroutine:table: 0x7fc2dfc074b0
pairs:function: 0x1081495b2
pcall:function: 0x108149635
ipairs:function: 0x1081493bb
select:function: 0x108149947
loadfile:function: 0x108149402
xpcall:function: 0x108149d29
```
:::

### 解説
Luaでは、グローバル変数はテーブル`_G`に保存されている。`for v in pairs(_G)`を利用して、存在するグローバル変数のキーを全て出力している。`ipairs`だと数字のkeyしか出力されないため、`pairs`が最適となる。

`getfiled`関数では、まず`string.gmatch`関数を用いて、単語毎に文字列を区切った配列でループをしている

Luaの正規表現はPerlと異なる点があるため、`[%w_]+`ついて解説する。`[]`は、setを表す。[0-9]や[a-Z]なら見覚えがあると思う。今回は全ての英数文字を示す`%w`と`_`が含まれる文字列がマッチされるようにしている。故にサンプルコードの出力結果は以下の通りとなる

```lua
for w in string.gmatch("aaa bb_b ccc_ __ddd", "[%w_]+") do print("[%w_]:" .. w) end
```

::: deatils 出力結果
```bash
[%w_]:aaa
[%w_]:bb_b
[%w_]:ccc_
[%w_]:__ddd
```
:::

## グローバル変数に値をセットする

```lua
local function setfield(f, v)
  local t = _G
  for w, d in string.gmatch(f, "([%w_]+)(%.?)") do
    if d == "." then
      t[w] = t[w] or {}
      t = t[w] -- tの参照を_G["t"] -> _G["t"]["x"]と代入する対象のフィールドに到達するまで、ネストしていく
    else
      t[w] = v
    end
  end
end
setfield("t.x.y", 10)
print(t.x.y)

-- 出力結果
-- 10
```

### 解説
もはや正規表現の解説でしかないが、`([%w_]+)(%.?)`という正規表現を利用している。

* `([%w_]+)`は、`_`を含む任意の英数字
* 丸括弧はグループ化。PCREと同じ
* `%.`は、`.`をエスケープしている
* `?`は最短マッチ。PCREと同じ

以上より、`_`を含む任意の英数字で`.`の手前までがグループ1、.から任意の文字列の手前がまでがグループ2となる。`string.gmatch`は、このグループ1とグループ2を、多値返却してくれるため、今回ケース場合以下のようにw,dに値が入ることとなる。

```bash
INPUT: t.x.y
1ループ目: w="t" d="."
2ループ目: w="x" d="."
3ループ目: w="x" d=""
```
3ループ目は、.がないのでグループ1しかマッチしないため、dは空文字列となる。`setfield`では、このdの値を元に最後のフィールドがどうかを判定している。

