---
title: "【Ruby】例外処理について掘り下げて考えた"
emoji: "⚠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "例外処理"]
published: true
published_at: 2024-01-15 12:00
---
## 自己紹介
はじめまして、はると申します。完全異業種からのエンジニア転職を目指して学習をしています。

## 概要
Rubyの基礎を学習しているとき、いくつか理解が難しかったところがありますが、そのうちの一つが「例外処理」でした😣
そのまま苦手だなと思いながら放置してしまっていたので、今回改めて学習しました。

## 注意
私は前職が完全異業種であり、英語も苦手で、スクールに入って初めてプログラミングに触れました。
そんな自分が理解しづらかった部分を、同じように初めてプログラミングの概念に触れた人に向けてまとめました。
自分の復習も兼ねて、超超噛み砕いて書いているため、周りくどい書き方になっている箇所もあるかと思います🙇

　　
## 例外処理とは
『例外処理（れいがいしょり、英語: exception handling）とは、IT業界で用いられる専門用語で、ある抽象レベルにおけるシステムの設計で想定されておらず、ユーザー操作によって解決できない問題に対処するための処理である。』[1)](#引用記事) とのことです。


## なぜ例外処理が必要なのか
予想外のこと（例外）が起こった時にプログラムが異常終了してしまうと、アプリが使えなくなったり、ユーザーに不安を与えてしまいます。
そのため、例外処理をして、例外が発生した時にも何らかの形で（エラーメッセージを表示してトップページに戻すなど）プログラムの実行を継続できるようにすることが必要と考えます。
　　
## 登場人物
- `raise`（`fail`も全く同じ動きをしますが、今回の記事では全てraiseで説明します）
- `begin`
- `rescue`
- `ensure`

実は、整理すると4つだけでした！！◎
1つずつ見ていきましょう。


## raise
`raise`は、例外を発生させるメソッドです。

```ruby
raise "例外です"
#=> Main.rb:1:in `<main>': 例外です (RuntimeError)
```

この１行のコードだけで、例外を発生させることができます。
`"例外です"`の部分を書き換えれば、好きな例外メッセージを表示させることができます。


## begin
`begin`、`rescue`、`ensure`は例外処理のための構文です。
1つずつどのような役割か見ていきます。

`begin`は、実行したいコードを書く部分です。例外が起こりうるコードの場合、`begin`〜`end`の例外処理の構文を使って書きます。

例えば、下記のようなメソッドがあった時、引数`x`には数値が渡されることを期待していますが、文字列が渡されるとエラーになってプログラムが終了してしまいます。（⚠️例外の発生）

```ruby
def addition(x)
  x = x + 1
  puts x
end

addition(2)   #=> 3

addition("2") #=> no implicit conversion of Integer into String (TypeError)
```
:::message alert
Rubyでは文字列と数値を足そうとするとエラーが起こります。
:::

　　
このような例外をキャッチするため、`begin`を使ってメソッドを書き換えていきたいです。
`rescue`の説明に移ります。


## rescue
`rescue`は、`begin`とセットで使われる構文です。
`begin`節の中で例外が発生したときには、例外が発生した時点で`begin`節内の処理を中断して`rescue`節に飛び、
`rescue`節の処理が実行されます。


```ruby
def addition(x)
  begin
    x = x + 1
    puts x
  rescue => evar   # everは省略してeと書くこともできます。また、どんな例外でもキャッチするようになっています。
    puts "例外が発生しました：#{evar.message}"  # begin内で例外が起こると直ちにrescue節に飛び、この行が実行される
  end
end

addition("2")
#=> 例外が発生しました：no implicit conversion of Integer into String
```
　　
これを工夫すると以下のような使い方ができます。
TypeError（型が違う）の時には、指定された型の値を入力するように促すメッセージを表示することができます。
少し実用的なイメージがついてきました！🐣

```ruby
def addition(x)
  begin
    x = x + 1
    puts x
  rescue TypeError => e   # TypeErrorの例外のみをキャッチするようにしました
    puts "xには数値を入力してください"
  end
end

addition("2")
#=> xには数値を入力してください
```
　　

:::message
余談：上記のコードでは、TypeErrorを指定したことで、TypeError以外の例外はキャッチされず強制終了してしまいます。
そのため、TypeError以外の例外が起こった時にもプログラムを終了せず継続させるようにするには、以下の例のように`rescue`節を追加します。
```diff_ruby
def addition(x)
  begin
    x = x + 1
    puts x
  rescue TypeError => e   # TypeErrorの例外のみをキャッチするようにしました
    puts "xには数値を入力してください"
+ rescue => e
+   puts "Error: 予期せぬエラーが発生しました - #{e.message}"
  end
end
```

Rubyでは、厳密にすべての例外をキャッチして対応するようにするには、
`rescue Exception => e`と書く必要がありますが、これでは、本来はプログラムを強制終了するべき致命的な例外の時も、プログラムを継続してしまいます。
しかし例のような `rescue => e` という書き方であれば、致命的な例外を除くすべての例外（StandardError）をキャッチしてくれます。
詳しくは[こちらの記事 2)](https://qiita.com/jnchito/items/dedb3b889ab226933ccf#exception%E3%82%92rescue%E3%81%99%E3%82%8B%E3%81%AE%E3%81%A7%E3%81%AF%E3%81%AA%E3%81%8Fstandarderror%E3%82%92rescue%E3%81%99%E3%82%8B)で紹介されていたのでご参照ください。
言語によっても、例外処理のアプローチが異なる、ということがわかり勉強になりました💡
:::


## ensure
簡単なコードの例外処理は`ensure`を使わなくても実行できるため、難しい方はこの部分は読み飛ばしても大丈夫です。
　　
`ensure`は、`begin`や`rescue`とセットで使われる構文です。
`ensure`節内に書かれたコードは、例外の有無にかかわらず、必ず実行されます。

実際に使うときの例としては、ファイルを読み込もうとするメソッドがあった時に、ファイルが存在しないなどの例外が発生すると、ファイルを開いている処理の途中でプログラムが終了してしまいます。
`ensure`は、何らかの理由で例外が起こった時でもファイルやネットワーク接続などを確実に終了するために使用されます。
　　
先ほどまでの例のコードは、`ensure`を使う必要のない簡単なコードですが、
コード例として、`ensure`を無理やり追加してみると以下のようになります。

```ruby
def addition(x)
  begin
    x = x + 1
    puts x
  rescue TypeError => e
    puts "xには数値を入力してください"
  ensure
    puts "この部分は例外が発生してもしなくても必ず実行されます"
  end
end

addition("2")
```

```:実行結果
xには数値を入力してください
この部分は例外が発生してもしなくても必ず実行されます
```


## raiseとbeginの構文の組み合わせの例
上記の例だと、`raise`、`begin`、`rescue`のそれぞれの役割や動きについてはわかりましたが、組み合わさるとよくわからなくなります😵‍💫

組み合わせる場合の例について考えてみます。
以下のコードでは、引数`x`が数値以外の時（`unless x.is_a?(Integer)`）には、意図的にTypeErrorを引き起こさせ`rescue`節に飛ぶようになっています。

```ruby
def addition(x)
  begin
    raise TypeError, "xには数値を入力してください" unless x.is_a?(Integer)

    x = x + 1
    puts x
  rescue TypeError => e
    puts e
  end
end

addition("2")
```
　　
簡単なコードなので、恩恵がわかりにくいですが、先ほどまでは`rescue`節を実行するタイミングは「例外が発生した時」でしたが、
`raise`を組み合わせることで、 **意図したタイミングで** 処理を中断し`rescue`節に飛ぶことができるようになりました。



## まとめ
- `raise`は、意図的に例外を発生させるメソッド。
- `begin`、`rescue`、`ensure`は例外処理のための構文。
- `begin`は、実行したいコードを書く部分で、`begin`節内で例外が発生したときに`rescue`節の処理が実行される。
- `ensure`節内に書かれたコードは、例外の有無にかかわらず、必ず実行される。
- `raise`と`begin`の構文を組み合わせることで、より意図的な動作を実現できる。

:::message
記事内ではすべて`begin`〜`end`で記載しましたが、
`begin`とセットじゃない`rescue`を使い省略形で書くことができます！
```ruby
def addition(x)
  x = x + 1
  puts x
rescue TypeError => e
  puts "xには数値を入力してください"
end
```
:::
　　

例外処理よくわからない〜が少し理解が進みました！！
ここまで読んでいただきありがとうございました🐥

　
## 参考・引用記事
#### 引用記事
1）[Wikipedia　例外処理](https://ja.wikipedia.org/wiki/%E4%BE%8B%E5%A4%96%E5%87%A6%E7%90%86)
2)[[初心者向け] RubyやRailsでリファクタリングに使えそうなイディオムとか便利メソッドとか　Exceptionをrescueするのではなく、StandardErrorをrescueする](https://qiita.com/jnchito/items/dedb3b889ab226933ccf#exception%E3%82%92rescue%E3%81%99%E3%82%8B%E3%81%AE%E3%81%A7%E3%81%AF%E3%81%AA%E3%81%8Fstandarderror%E3%82%92rescue%E3%81%99%E3%82%8B)

#### 参考記事
- [IT用語辞典 e-Words　例外処理](https://e-words.jp/w/%E4%BE%8B%E5%A4%96%E5%87%A6%E7%90%86.html)
- [「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典　例外処理](https://wa3.i-3-i.info/word1427.html)
- [Ruby初心者必見！エラー処理の基本「raise」を完全理解する5ステップ](https://jp-seemore.com/web/9618/)
- [Ruby 3.3 リファレンスマニュアル　Kernel.#fail](https://docs.ruby-lang.org/ja/latest/method/Kernel/m/fail.html)
- [Ruby 3.3 リファレンスマニュアル　制御構造　raise](https://docs.ruby-lang.org/ja/latest/doc/spec=2fcontrol.html#raise)
- [Ruby 3.3 リファレンスマニュアル　制御構造　begin](https://docs.ruby-lang.org/ja/latest/doc/spec=2fcontrol.html#begin)
- [beginとセットじゃないrescueって何なの？](http://blog.livedoor.jp/sasata299/archives/51338814.html)
