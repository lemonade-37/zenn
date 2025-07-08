---
title: "【Ruby】attr_accessor・ゲッター・セッターについて掘り下げて考えた"
emoji: "🥟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby"]
published: true
published_at: 2024-01-12 12:00
---
## 自己紹介
はじめまして、はると申します。完全異業種からのエンジニア転職を目指して学習をしています。

## 概要
Rubyの `attr_accessor` について、わかりやすく解説されている記事はすでにたくさんありますが、自分が理解できるまで掘り下げたことをまとめました。

## 注意
私は前職が完全異業種であり、スクールに入って初めてプログラミングに触れました。
そんな自分が理解しづらかった部分を、同じように初めてプログラミングの概念に触れた人に向けてまとめました。
自分の復習も兼ねて、超超噛み砕いて書いているため、周りくどい書き方になっている箇所もあるかと思います🙇


## ゲッターとセッターとは
こちらの記事が大変わかりやすかったので引用させていただきます🙇

https://qiita.com/k-penguin-sato/items/5b75be386be4c55e3abf

こちらの記事で全て理解できた方は今回の記事は対象外となります。
　　
一応、公式リファレンスからも引用すると、それぞれ下記のように解説されています。
`attr_reader`：インスタンス変数 name の**読み取りメソッド**を定義します。[1)](#引用記事)
`attr_writer`：インスタンス変数 name への **書き込みメソッド (name=)** を定義します。[2)](#引用記事)
`attr_accessor`：インスタンス変数 name に対する **読み取りメソッドと書き込みメソッドの両方** を定義します。[3)](#引用記事)



## 噛み砕いて考える
ゲッターとセッターについてはなんとなくわかりましたが、以下の疑問が残りました。
- ゲッターを追加した後に、それを呼び出すときに使われているメソッド名はどの部分で決めた名前が使われている？
- ゲッターとセッターの解説内によく出てくる`initialize`メソッドはなぜ必要？
- 実行順はどうなっている？😵‍💫

　　
この疑問を解決するため、どのような順番でコードが実行されているのか考えてみました。
下記のRubyのコードがあったとして、実行順に見てみます。
```ruby
class MeatBun            # 肉まんクラス
  attr_reader :filling
  attr_writer :filling
  def initialize(filling)
    @filling = filling   # fillingは具
  end
end

bun1 = MeatBun.new('豚肉')
bun1.filling = '鶏肉'
p bun1.filling
```
　　
1. `bun1 = MeatBun.new('豚肉')`実行時
    MeatBunクラスの`initialize`メソッド（新しいインスタンスが作成されたときに自動的に呼び出されるメソッド）が呼び出されます。
    `initialize`メソッドの引数として`'豚肉'`が渡されて、`@filling`に`'豚肉'`が入っている状態でMeatBunのインスタンスが生成され、それが`bun1`に代入されています🐖
    この段階では`attr_reader`や`attr_writer`は介していません。
    　　
2. `bun1.filling = '鶏肉'`実行時
    `attr_writer` により生成された `filling=(val)` メソッドが呼び出されます。
    元々`'豚肉'`が入っていた`@filling`に、引数で渡した`'鶏肉'`を再代入します🐓
    　　
    下記のメソッドが、`attr_writer :filling`の１行で表現されているということです。
    （`val`という言葉を使っているのは、公式リファレンスの通りにしました。）
    ```ruby
    def filling=(val)
      @filling = val
    end
    ```
    :::message
    `filling`の部分は、この時指定した名前になります。
    本来は`initialize`で定義したインスタンス変数名と違う名前にすることはないと思いますが、
    どこで命名したものが、どこに使われるのか確認するため、あえて`attr_writer :croissant`と書いた場合には、下記のような`croissant=(val)` メソッドとなります🥐
    :::
    ```ruby
    def croissant=(val)
      @croissant = val
    end
    ```
3. `p bun1.filling`実行時
    `attr_reader`により生成された`filling`メソッドが呼び出されます。
    そして`@filling`が取得されます。
    先ほど`@filling`の中に、`'鶏肉'`を再代入していたため、`'鶏肉'`が表示されます。
    　　
    下記のメソッドが、`attr_reader :filling`の１行で表現されているということです。
    ```ruby
    def filling
      @filling
    end
    ```
    :::message
    こちらも先ほどと同様、本来`attr_writer`や`initialize`と違う名前にすることはないと思いますが、
    `attr_reader :filling`の部分で命名した名前のメソッドとなります。
    :::

## attr_accessorについて
上記のように`attr_reader`と`attr_writer`を両方使う場合は、`attr_accessor`の１行で済みます。
```ruby
class MeatBun
  attr_accessor :filling  # ここ
  def initialize(filling)
    @filling = filling
  end
end
```



## 疑問は解決できたか？
- `bun1.filling = '鶏肉'`で使われていた`.filling`の部分は、`attr_writer`で生成された`filling=(val)`メソッドのことでした。
- `initialize`メソッドがないとインスタンス生成時には`@filling`が無いので、`bun1 = MeatBun.new('豚肉')`のように、初めから値を渡してインスタンス作成することはできません。
ですが、`initialize`が無くても、`bun1 = MeatBun.new`と引数無しでインスタンスを生成することはできます。`attr_writer`と`attr_reader`があれば、引数無しでインスタンス生成後に`bun1.filling = '鶏肉'`や`p bun1.filling`で値の代入や参照が可能です。
- 実行順番を整理したことで、
最初に`@filling`への値の代入を行っているのは`initialize`メソッド、
    値の更新を行っているのは`attr_writer`で定義されたメソッド、
    値を参照しているのは`attr_reader`で定義されたメソッド、
    とそれぞれ別々のメソッドで行われていることがわかりました。
- また、下記の3つのメソッドでは同じ名前のインスタンス変数`@filling`が使われていますが、同じクラス内の異なるメソッドで同じ名前のインスタンス変数を使用している場合、それらが同じものと見なされるということがわかりました。（だから値の更新や参照をしても同じ`@filling`に対して行えます）
    ```ruby
    class MeatBun
      def filling        # attr_reader :filling
        @filling
      end

      def filling=(val)  # attr_writer :filling
        @filling = val
      end

      def initialize(filling)
        @filling = filling
      end
    end
    ```

## さいごに
細かく噛み砕いたことでずっと苦手意識を持っていたゲッターやセッターの概念、`attr_accessor`についての理解が進みました！
ここまで読んで頂きありがとうございました🙂

　
## 参考・引用記事
#### 引用記事
1）[Ruby 3.3 リファレンスマニュアル attr_reader](https://docs.ruby-lang.org/ja/latest/method/Module/i/attr_reader.html)
2）[Ruby 3.3 リファレンスマニュアル attr_writer](https://docs.ruby-lang.org/ja/latest/method/Module/i/attr_writer.html)
3）[Ruby 3.3 リファレンスマニュアル attr_accessor](https://docs.ruby-lang.org/ja/latest/method/Module/i/attr_accessor.html)

#### 参考記事
- [【Ruby】「ゲッター」と「セッター」を理解する](https://qiita.com/k-penguin-sato/items/5b75be386be4c55e3abf)
- [Ruby 3.3 リファレンスマニュアル initialize](https://docs.ruby-lang.org/ja/latest/method/Object/i/initialize.html)
