---
title: "ziglingsやってみる(随時更新)"
type: "note"
description: "ziglingsの翻訳、所感を随時更新でつらつらと書きます"
thumbnail: ""
publish: true
tags:
  - "tech/zig"
  - "tech/開発環境"
---

# はじめに

[zigling](https://github.com/ratfactor/ziglings) というZigのチュートリアルリポジトリがあるので、単なる翻訳+所感をつらつら書いてきます。
リンクのソースが変わっていた場合は、[fork版](https://github.com/shuntaka9576/ziglings)を参照ください。

# 001
[001_hello.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/001_hello.zig)

`Hello World!`と標準出力させる問題。ポイントは以下。

* Zig関数はデフォルトでプライベート関数
* main関数はパブリックである必要があり

接頭辞に`pub`キーワードがあるかどうかで、公開/非公開を制御可能
```zig:非公開関数
fn foo() void {
}
```

```zig:公開関数
pub fn foo() void {
}
```

## 所感
print関数の`.{}`の部分がなぜ必要なのかよくわからない
=> 003で解決。第二引数には匿名リストリテラルを指定する必要あり、表示したい変数がない場合は本例のように空で指定する

```zig
  std.debug.print("hello wolrd", .{});
```

# 002
[002_std.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/002_std.zig)

Zigには`@import`という組み込み関数がある。以下のようにインポートする名前と同名の定数値として保存することが推奨される。

```zig
const std = @import("std")
```

* importsは定数として宣言する必要がある
  * importsは実行時ではなくコンパイル時に利用される関数であるため
* Zigは定数値をコンパイル時に評価する


参考
* [struct definition with var instead of const in zig language回答部](https://stackoverflow.com/a/62567550/695615)


# 003
[003_assignment.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/003_assignment.zig)

constは不変、varは可変

|値型|説明|
|---|---|
|u8|符号なし型8ビット整数(0 to 256)|
|i8|符号あり型8ビット整数(-128 to 127)|
|u32|符号あり型32ビット整数(0 to 4,294,967,295)|
|i64|符号あり型32ビット整数(−9,223,372,036,854,775,808 to 9,223,372,036,854,775,807)|

```zig:宣言例
// 変更不可
const v1: u8 = 20;
const v2: i8 = -20;
// 可変
var v3: u8 = 20;
```

```zig:回答例
const std = @import("std");

pub fn main() void {
  var n: u8 = 50;
  // constなので再代入不可
  // n = n + 5;
  n = n + 5;


  // u8の範囲は、0 - 256なのでコンパイルエラー
  // const pi: u8 = 314159;
  const pi: u32 = 314159;
  // u8の範囲は、0 - 256なのでコンパイルエラー
  // const negative_eleven: u8 = -11;
  const negative_eleven: i8 = -11;

  std.debug.print("{} {} {}\n", .{n, pi, negative_eleven});
}
```

* `std.debug.print`の第2パラーメーターは、匿名リストリテラルと呼ぶ
  * 001の時点で疑問だった第二引数の意味が分かった

## 所感
数値型の宣言方法は、Rustとほぼ一緒かな。


# 004
[004_arrays.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/004_arrays.zig)

配列の宣言

```zig:配列長を指定して宣言
var foo: [3]u32 = [3]u32{ 42, 108, 5423}
```

```zig:コンパイル時に配列長が確定している場合省略可(基本こっちを利用するかな)
var foo = [_]u32{ 42, 108, 5423}
```

配列は、メジャー言語と同じで0始まり。
```zig:要素の取得
const foo = foo[2] // 5432
```

ここら辺は、003でやった通り。配列だとconstでも挙動が違う言語もあり、TSは配列の要素は再代入できたりする。zigでは再代入不可。
```zig:配列の値変更
// varな配列では変更可
var var_array = [_]u8{1, 2, 3};
var_array[1] = 24;

// constな配列の要素に再代入は不可
// const const_array = [_]u8{1, 2, 3};
// const_array[1] = 24;
```

```zig:配列長の取得
const length = some_primes.len;
```


# 005
[005_arrays2.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/005_arrays2.zig)

`++`演算子で配列の結合が可能
```zig:配列の結合
const le = [_]u8{ 1, 3 };
const et = [_]u8{ 3, 7 };

const leet = le ++ et ++ [_]u8{ 5 };
std.debug.print("{}", .{ leet[4] }); // 5
```

`**`演算子で配列の繰り返しが可能。あると便利だがユースケースそんなにあるかなぁ..
```zig:配列の繰り返し結合
const bit_pattern = [_]u8{ 1, 0, 0, 1 } ** 3;
std.debug.print("bit_pattern len: {}\n", .{ bit_pattern.len }); // 11
std.debug.print("bit_pattern last value: {}\n", .{ bit_pattern[bit_pattern.len - 1] }); // 1
```

`for of`のような形で配列の添字使用しない形でのループ記法がサポートされている。ここら辺はC言語よりモダンで良さそう
```zig:配列のループ
std.debug.print("LEET:", .{});

for (leet) |n| {
    std.debug.print("{}", .{n});
}

std.debug.print(", Beits:", .{});

for (bit_pattern) |n| {
    std.debug.print("{}", .{n});
}
// LEET:1337, Beits:100110011001
```

# 006

[006_strings.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/006_strings.zig)

文字列に関する例題

* Zigは文字列をバイトの配列として格納

```zig:文字列をバイト配列で格納している例
const std = @import("std");

pub fn main() void {
    const helloStr = "Hello";
    // [_]u8{ 'H', 'e', 'l', 'l', 'o' };と同様

    for (helloStr) |n| {
        std.debug.print("{}, ", .{n});
    }
}
```

```bash:出力結果
72, 101, 108, 108, 111,
```

`005`で行った配列操作を、文字列が配列であることから活用可能。`std.debug.print`で数値出力
```zig:例題
const std = @import("std");

pub fn main() void {
    const ziggy = "stardust";

    // const d: u8 = ziggy[????]
    const d: u8 = ziggy[4];

    // const laugh = "ha" ???;
    const laugh = "ha" ** 3;

    const major = "Major";
    const tom = "Tom";
    // const major_tom = major ??? tom;
    const major_tom = major ++ tom;

    // `{}`プレースホルダーを指定している。
    std.debug.print("d={u} {s}{s}\n", .{ d, laugh, major_tom });
}
```

`u`は、UTF-8文字。`s`はUTF-8文字列としてフォーマットするように指示している。`u`を指定しない場合は、UTF-8に対応する10進数である`100`が表示される。
`s`を指定しない場合、コンパイルエラーとなる。理由は、`変数d`はu8型だったのに対し、`変数laughと変数major_tom`はu8配列であるため、配列のフォーマットを試みるためである。
```bash:出力結果
d=d hahahaMajorTom
```

`u`の代わりにASCIIエンコード文字をフォーマットする`c`フォーマット文字列も存在する。UTF-8の最初128文字はASCIIと同様なので同じように動作する。

# 007

[007_strings2.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/007_strings2.zig)

文字列関する例題2

`\\`を先頭につけることで、改行を含む文字列を定義可能。視認性が高く良いと思う。

```zig
const std = @import("std");

pub fn main() void {
    const lyrics =
        \\Ziggy played guitar
        \\Jamming good with Andrew Kelley
        \\And the Spiders from Mars
    ;

    std.debug.print("{s}", .{lyrics});
}
```

```bash:出力結果
Ziggy played guitar
Jamming good with Andrew Kelley
And the Spiders from Mars
```

# 008
初のクイズタイム。書き換えたところはコメントにしている。

```zig
const std = @import("std");

pub fn main() void {
    const letters = "YZhifg";

    // const x: usize = 1;
    var x: usize = 1;

    var lang: [3]u8 = undefined;

    lang[0] = letters[x];

    x = 3;
    // lang[???] = letters[x];
    lang[1] = letters[x];

    // x = ???;
    x = letters.len - 1;
    // lang[2] = letters[???];
    lang[2] = letters[x];

    // We want to "Program in Zig!" of course:
    std.debug.print("Program in {s}!\n", .{lang});
}
```


```bash:出力結果
Program in Zig!
```

今までの課題の既出でないポイントとしては、`undefined`でメモリ(空配列)を初期化できる点が重要だと思う。


# 009

[009_if.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/009_if.zig)

Zigのifで重要な点は、唯一ブール値を受け入れる点。その他の記法はよく利用される言語と似ている。

* a == b   means "a equals b"
* a < b    means "a is less than b"
* a > b    means "a is greater than b"
* a != b   means "a does not equal b"

```zig
const std = @import("std");

pub fn main() void {
    const foo = 1;

    // if (foo) { // bool値出ないためコンパイルエラー
    if (foo == 1) {
        std.debug.print("Foo is 1!\n", .{});
    } else {
        std.debug.print("Foo is not 1!\n", .{});
    }
}
```

```bash:出力結果
Foo is 1!
```


# 010

[010_if2.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/010_if2.zig)

ifを利用した三項演算子チックな書き方が可能


```zig
const std = @import("std");

pub fn main() void {
    var discount = true;

    // var price: u8 = if ???;
    var price: u8 = if (discount) 17 else 18;

    std.debug.print("With the discount, the price is ${}.\n", .{price});
}
```

```bash:出力結果
With the discount, the price is $17.
```

# 011

[011_while.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/011_while.zig)

ifの判定時同様にwhileにはbool値を指定する必要あり、他言語同様継続条件を指定する。

```zig
const std = @import("std");

pub fn main() void {
    var n: u32 = 2;

    // while (???) {
    while (n != 10) {
        std.debug.print("{}\n", .{n});

        n += 2;
    }

    std.debug.print("n={}\n", .{n});
}
```

# 012

[012_while2.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/012_while2.zig)

Zigのwhileステートメントには、`continue式`を指定可。
```zig
while (condtion): (continue expresion) {
  ...
}
```

whileループの以下のタイミングで実行される

* ループの終わり
* 明示的にcontinueが実行された

```zig
const std = @import("std");

pub fn main() void {
    var n: u32 = 2;

    // while (n < 1000): ??? {
    while (n < 1000) : (n += 2) {
        std.debug.print("{} ", .{n});
    }

    std.debug.print("n={}\n", .{n});
}
```

```bash:出力結果
2 4 6 8 10 12 14 16 18 20 22 24 26 28 30 32 34 36 38 40 42 44 46 48 50 52 54 56 58 60 62 64 66 68 70 72 74 76 78 80 82 84 86 88 90 92 94 96 98 100 102 104 106 108 110 112 114 116 118 120 122 124 126 128 130 132 134 136 138 140 142 144 146 148 150 152 154 156 158 160 162 164 166 168 170 172 174 176 178 180 182 184 186 188 190 192 194 196 198 200 202 204 206 208 210 212 214 216 218 220 222 224 226 228 230 232 234 236 238 240 242 244 246 248 250 252 254 256 258 260 262 264 266 268 270 272 274 276 278 280 282 284 286 288 290 292 294 296 298 300 302 304 306 308 310 312 314 316 318 320 322 324 326 328 330 332 334 336 338 340 342 344 346 348 350 352 354 356 358 360 362 364 366 368 370 372 374 376 378 380 382 384 386 388 390 392 394 396 398 400 402 404 406 408 410 412 414 416 418 420 422 424 426 428 430 432 434 436 438 440 442 444 446 448 450 452 454 456 458 460 462 464 466 468 470 472 474 476 478 480 482 484 486 488 490 492 494 496 498 500 502 504 506 508 510 512 514 516 518 520 522 524 526 528 530 532 534 536 538 540 542 544 546 548 550 552 554 556 558 560 562 564 566 568 570 572 574 576 578 580 582 584 586 588 590 592 594 596 598 600 602 604 606 608 610 612 614 616 618 620 622 624 626 628 630 632 634 636 638 640 642 644 646 648 650 652 654 656 658 660 662 664 666 668 670 672 674 676 678 680 682 684 686 688 690 692 694 696 698 700 702 704 706 708 710 712 714 716 718 720 722 724 726 728 730 732 734 736 738 740 742 744 746 748 750 752 754 756 758 760 762 764 766 768 770 772 774 776 778 780 782 784 786 788 790 792 794 796 798 800 802 804 806 808 810 812 814 816 818 820 822 824 826 828 830 832 834 836 838 840 842 844 846 848 850 852 854 856 858 860 862 864 866 868 870 872 874 876 878 880 882 884 886 888 890 892 894 896 898 900 902 904 906 908 910 912 914 916 918 920 922 924 926 928 930 932 934 936 938 940 942 944 946 948 950 952 954 956 958 960 962 964 966 968 970 972 974 976 978 980 982 984 986 988 990 992 994 996 998 n=1000
```


# 013

[013_while3.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/013_while3.zig)

continue式は、012で説明したcontinue式が実行される点で他に特有な点はない。

```zig
const std = @import("std");

pub fn main() void {
    var n: u32 = 1;

    while (n <= 20) : (n += 1) {
        // if (n % 3 == 0) ???;
        if (n % 3 == 0) continue;
        // if (n % 5 == 0) ???;
        if (n % 5 == 0) continue;

        std.debug.print("{} ", .{n});
    }

    std.debug.print("\n", .{});
}
```


# 014
[014_while4.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/014_while4.zig)

こちらも特有な点はなし

```zig
const std = @import("std");

pub fn main() void {
    var n: u32 = 1;

    while (true) : (n += 1) {
        // if (???) ???;
        if (n == 4) break;
    }

    std.debug.print("n={}\n", .{n});
}
```

# 015
[015_for.zig](https://github.com/ratfactor/ziglings/blob/main/exercises/015_for.zig)

`005`でも登場していた配列長を取らなくても終端までループできるfor記法

```zig
const std = @import("std");

pub fn main() void {
    const story = [_]u8{ 'h', 'h', 's', 'n', 'h' };

    std.debug.print("A Dramatic Story:", .{});

    for (story) |scene| {
        if (scene == 'h') std.debug.print(":-) ", .{});
        if (scene == 's') std.debug.print(":-( ", .{});
        if (scene == 'n') std.debug.print(":-| ", .{});
    }

    std.debug.print("The End.\n", .{});
}
```

```bash:出力結果
A Dramatic Story::-) :-) :-( :-| :-) The End.
```

# 016

forループ時の添字取得方法

```zig
const std = @import("std");

pub fn main() void {
    const bits = [_]u8{ 1, 0, 1, 1 };
    var value: u32 = 0;

    for (bits, 0..) |bit, i| {
        const i_u32: u32 = @intCast(i)
        var place_value = std.math.pow(u32, 2, i_u32);
        value += place_value * bit;
        std.debug.print("bit: {}, index:{}, place_value: {}\n", .{ bit, i, place_value });
    }

    std.debug.print("The value ob bits '1101': {}.\n", .{value}); // 1+4+8 = 13
}
```

累乗する関数も初出です。

`Returns x raised to the power of y (x^y).`(x を y のべき乗にした値 (x^y) を返す。)

```zig
// i^2を表す
var place_value = std.math.pow(u32, 2, @intCast(u32, i));
```
