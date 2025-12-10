---
title: "Marshal.loadのときに現れる謎のtrueがなんなのかわかった"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Ruby]
published: true
published_at: 2025-12-12 00:00
---

Marshal という Ruby オブジェクトとバイト列を変換するモジュールで文字を保存、復元したときに出てくる謎の `true` について、何を示すものなのかと狙いがわかったのでまとめました。

![](/images/mysterious_true_in_marshalload.png)

# `Marshal` というモジュール

> Ruby オブジェクトをファイル(または文字列)に書き出したり、読み戻したりする

[Marshal というモジュール](https://docs.ruby-lang.org/ja/latest/class/Marshal.html)があります。

[Marshal.load](https://docs.ruby-lang.org/ja/latest/class/Marshal.html#M_LOAD) はデータを読み込んで、元のオブジェクトと同じ状態をもつオブジェクトを生成します。
そのとき、第二引数に proc を渡すと読み込んだオブジェクトを引数にその手続きを呼びだします。

具体的な内容だと理解しやすいと思うので、例をあげます。
以下では `[:a, :b, :c]` を `Marshal.dump` で文字列化し [^1] 、その結果を `Marshal.load` で読み込んでいます [^2]。 `->(x) { p x }` の内容が 4 回、`:a`、`:b`、`:c`、`[:a, :b, :c]` と出力されているのがわかります。

```ruby
irb --simple-prompt
>> dumped = Marshal.dump([:a, :b, :c])
=> "\x04\b[\b:\x06a:\x06b:\x06c"
>> Marshal.load(dumped, ->(x) { p x })
:a
:b
:c
[:a, :b, :c]
=> [:a, :b, :c]
```

# `Marshal.load` で出てくる `true`

先日この `Marshal.load` で String を扱うときだけ、文字本体だけでなく `true` が proc に渡ってくるのを知りました。
他の `true`、 `1`、 `:a` のときはその要素しか渡ってきていないのに対し、 `"a"` のときは `true` と `"a"` が渡ってきているのに注目してください。

```ruby
irb --simple-prompt
>> Marshal.load(Marshal.dump(true), ->(x) { p x })
true
=> true
>> Marshal.load(Marshal.dump(1), ->(x) { p x })
1
=> 1
>> Marshal.load(Marshal.dump(:a), ->(x) { p x })
:a
=> :a
>> Marshal.load(Marshal.dump("a"), ->(x) { p x })
true
"a"
=> "a"
```

この `true` が何なのか、みなさんはわかりますか？私はわかりませんでした。

# 文字列を `Marshal.dump` するときは、文字とエンコーディングのペアが保存されている

調べたところ、文字の場合はエンコーディングと文字のペアで値が保存されるんですね。
他の文字エンコーディングの場合は、文字列がそのまま表示されます。
`UTF-8` と `ascii` のみ特殊で、それぞれ `true` と `false` に対応しています。

```ruby
irb --simple-prompt
>> Marshal.load(Marshal.dump("a".encode(Encoding::EUCJP)), ->(x) { p x })
"EUC-JP"
"a"
=> "a"
>> Marshal.load(Marshal.dump("a".encode(Encoding::SHIFT_JIS)), ->(x) { p x })
"Shift_JIS"
"a"
=> "a"
>> Marshal.load(Marshal.dump("a".encode(Encoding::UTF_8)), ->(x) { p x })
true
"a"
=> "a"
>> Marshal.load(Marshal.dump("a".encode(Encoding::ASCII)), ->(x) { p x })
false
"a"
=> "a"
```

# `UTF-8` と `US-ASCII` だけエンコーディング情報を boolean にしている理由

なぜ `UTF-8` と `US-ASCII` のみ特殊で、それぞれ `true` と `false` になっているんでしょうか？
ruby-jp の #random チャンネルに投げかけたところ mame さんと chiastolite さんが教えてくれました。^[https://ruby-jp.slack.com/archives/CLTRGLV4Z/p1763342261079589] ^[https://ruby-jp.slack.com/archives/CLTRGLV4Z/p1763349550688859]

https://github.com/ruby/ruby/commit/7afe0c92ea9f4e363429f3209567b4bc3de302db でこの変更が入っていて

> more compact encoding information for US-ASCII and UTF-8.

であるそうです。^[最近のコードだと https://github.com/ruby/ruby/blob/4870fbd04a015717800a8b5ab87da03e4d93e894/marshal.c#L664-L670]
私は日本語を母語としており Ruby でも UTF8 環境でのプログラミングをしていました。だから偶然 `true` が出てきて戸惑ったんですね。
最近の日本語を母語としている Ruby プログラマの多くの環境では UTF8 がデフォルトになっているんじゃないですかね。みなさんの所でも同じようなことが起きると思います。

# どのくらいお得か

ただエンコード情報を `"UTF-8"` から `true` に変えてそんなに嬉しいんでしょうかね？
もちろん数バイトでも小さいと嬉しいでしょうけれども、数バイトと引き換えにハードコードすることで柔軟性やわかりやすさが失われていませんかね。どのくらい効果が見込めるんでしょうか。
文字列のエンコード情報は、実は文字要素毎につきます。例えば配列の要素に文字列があるときは、要素毎にエンコーディング情報が付与されています。

```ruby
irb --simple-prompt
?> array = [
?>   "foo".encode(Encoding::UTF_8),
?>   "bar".encode(Encoding::ASCII),
?>   "baz".encode(Encoding::EUCJP),
?>   "qux".encode(Encoding::SHIFT_JIS)
>> ]
=> ["foo", "bar", "baz", "qux"]
>> Marshal.load(Marshal.dump(array), ->(x) { p x })
true
"foo"
false
"bar"
"EUC-JP"
"baz"
"Shift_JIS"
"qux"
["foo", "bar", "baz", "qux"]
=> ["foo", "bar", "baz", "qux"]
```

なので、文字列の要素がたくさんあるものを Marshal.dump したときのサイズが結構変わるんですね。
試しに 1,000 個の `"a"` が UTF8 のときと EUC_JP のときに Marshal.dump したときのバイトサイズを比べてみましょう。

```ruby
irb --simple-prompt
>> utf8_array = 1000.times.map { "a".encode(Encoding::UTF_8) }
=>
["a",
...
>> eucjp_array = 1000.times.map { "a".encode(Encoding::EUCJP) }
=>
["a",
...
>> Marshal.dump(utf8_array).bytesize
=> 8007
>> Marshal.dump(eucjp_array).bytesize
=> 9020
```

9020/8007 = 1.127 と 10% 以上の差になっています。
このように最も使われるであろうエンコーディングの 2 つ、ascii と utf8 だけを特別扱いして Marshal.dump したときのバイトサイズを小さく保っているんですね。
このくらいサイズに違いが出るなら柔軟性やわかりやすさとのトレードオフでこういった仕様を取り入れたくなる理由もうなずけますね。

# まとめ

`Marshal.load` の第二引数に proc を渡して、バイト列から Ruby のオブジェクトへの変換を観察しているときに出てきて不可解だった `true` は、文字のエンコーディングが UTF-8 であることを表しており、文字のエンコーディングを `"UTF-8"` と保存するかわりに `true` にしておくことが実験した場合だと 10% 程度のバイトサイズ差になることがわかりました。

[^1]: 文字列（バイト列）の仕様は https://docs.ruby-lang.org/en/master/language/marshal_rdoc.html に説明があります。
[^2]: この例だと文字列化の有り難みが薄いですが、ファイルに書き出したり、別のシステムと通信するときに便利な仕組みです。
