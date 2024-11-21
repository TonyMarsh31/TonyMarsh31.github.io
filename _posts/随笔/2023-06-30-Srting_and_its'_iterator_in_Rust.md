---
layout: post
title: String and its' iterator in Rust
date: 2023-06-30 18:26 +0800
category: [ 随笔 ]
tag: [ Rust,String ]
---

a note of what i learned from this video:

{% include embed/youtube.html id='Mcuqzx3rBWc' %}

## prelude

rust中对于String的概念与操作很复杂， \
这主要是因为String本身就是一个高度抽象的复杂概念了，\
而rust作为一个lowlevel 语言，对于String的操作在表述上可以/必须更精确 \
that's the feature ,是语言表达力的一部分。

## few about utf-8 encoding rules

一个utf-8 字符 由1~4个byte组成

一些简单的规则:
第一个byte 前几位表示这个字符有几个byte组成 \
如果以0开头则表示只有当前一个byte表示字符，这个规则使得utf-8兼容ASCII \
如果以11开头则表示这个字符用两个byte表示\
111开头 → 三个byte  1111开头 → 四个byte

如果一个字符由多个byte组成，那么后续的byte均由10开头
这样即可区分当前byte是否是一个字符的中间部分

![what_is_a_crab_emoji_made_of](https://github.com/TonyMarsh31/picx-images-hosting/raw/master/Blog/String_in_Rust/what_is_a_crab_emoji_made_of.1ap0xa5wty.png)

## glossary of terms

![scalar-value-take-up-only-part-of-the-whole-Unicode-codespace](https://github.com/TonyMarsh31/picx-images-hosting/raw/master/Blog/String_in_Rust/scalar-value-take-up-only-part-of-the-whole-Unicode-codespace.2yyduhwad1.png)

+ Unicode codespace: A range of integers from 0 to 10FFFF16.
+ Code point: Any value in the Unicode codespace.
+ Unicode scalar value: Any Unicode code point except high-surrogate and low-surrogate code points.

high-surrogate and low-surrogate code points: \
高代理值和低代理值本身没有实际意义，它们只能在UTF-16中成对使用，\
因此Rust认为它们不是有效的Unicode字符。

可以理解为 rust 是(utf-8/ unicode scalar value) supprted, 但不支持所有的 UTF字符集中的字符

## rustlang 中的String

rust中 char 表示一个 Unicode scalar value \
所以我们可以做到 print! emoji

```rust
println!("{}", '🦀');
```

而因为utf-8不定长字符的特性, String 在lowlevel上不能简单理解为char[]数组来使用(除非你确定你要这么做)，\
如果你想要以一个高抽象度的视角出发，以'字符'为单位遍历或slice一个String，那么你不能：

+ 直接通过index操作String，
+ 通过.chars()方法返回的迭代器操作String

这是因为前者以byte为单位，后者以Unicode scalar value为单位，\

而一个byte不一定就表示一个字符。(绝大部分中文字符在utf-8)中都是3个byte表示的)\
且一个Unicode scalar value 也不一定就是一个字符\

(取决于font 与 terminal,下面的字符可能无法正常显示为一个整体的字符)\
例如👨🏾 = 👨 + 🏾  , 或者  कां = क + ा + ं\
(这里的`+`表示 字符的binnary的直接连接)

![有些字符需要多个unicode标量来表意](https://github.com/TonyMarsh31/picx-images-hosting/raw/master/Blog/String_in_Rust/有些字符需要多个unicode标量来表意.4g4iwajgk8.png)

综上， 如果不想要以lowlevel的方式操作String，\
而是要以字符为单位操作String，那么你应该使用unicode-segmentation crate \
用 `.graphemesll`获取 字符素集群的迭代器

```rust
use unicode_segmentation::UnicodeSegmentation;

fn main() {
   for g in  "👨🏾,कां".graphemes(true){
    println!("{}\n", g);
   }
}
```

>
取决于font 与 terminal, 上面print宏的结果（以及代码本身）可能无法正常显示为一个整体的字符\
但是可以同时print一个计数器，可以验证通过`.graphemes`,仅迭代了两次)
{:.prompt-info}
  
